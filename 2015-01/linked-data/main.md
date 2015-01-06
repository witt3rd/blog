# Overview

Our goal is to use the [PEG-based parser](http://en.wikipedia.org/wiki/Parsing_expression_grammar) [Parboild2](https://github.com/sirthias/parboiled2) in order to parse [N-Triples](http://www.w3.org/2001/sw/RDFCore/ntriples) documents as part of [Maana’s](http://maana.io) support for [Linked Data](http://linkeddata.org).

Our test project for this article can be found [here](https://github.com/witt3rd/linked-data). I won’t bother going into the setup or basic concepts of PEGs or PB2, since they are all covered well enough on the PB2 Github page. Instead, we are going to just dive right in building our grammar and parsing some sample files.

## N-Triples

The N-Triples format is dead simple: a series of lines consisting of a {subject, predicate, object} triple followed by a period. For example, the following are a few random lines of [OWL](http://en.wikipedia.org/wiki/Web_Ontology_Language):

~~~~ { .xml }
<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2000/01/rdf-schema#label> "Concept"@en .
<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2000/01/rdf-schema#isDefinedBy> <http://www.w3.org/2004/02/skos/core> .
<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2004/02/skos/core#definition> "An idea or notion; a unit of thought."@en .
<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <http://www.w3.org/2002/07/owl#Class> .
_:BX2Db3de8bfX3A149861d9206X3AX2D7ffe <http://www.w3.org/1999/02/22-rdf-syntax-ns#first> <http://www.w3.org/2004/02/skos/core#Concept> .
_:BX2Db3de8bfX3A149861d9206X3AX2D7ffe <http://www.w3.org/1999/02/22-rdf-syntax-ns#rest> _:BX2Db3de8bfX3A149861d9206X3AX2D7ffd .
~~~~

There are three types of values that comprise a triple: urirefs, namedNodes, and literals. The BNF lends itself straightforwardly to our parse rules.

## Baby Step 0

The first decision we need to make is how we will feed text to our parser: the whole document or line-by-line. As some of the datasets we wish to ingest can be quite large (e.g., [YAGO](http://www.mpi-inf.mpg.de/departments/databases-and-information-systems/research/yago-naga/yago/) weighs in at 18.5GB uncompressed), it seems to make more sense to parse line-by-line.

Just to get things started, let’s create a skeleton of our parser in its own module (nTriples.scala), sketching in the BNF from the [spec](http://www.w3.org/2001/sw/RDFCore/ntriples) and defining our output — a triple.

For now, we’ll just parse a line (string) and output a `Triple` consisting of the string repeated for subject, predicate, object, just to keep things simple and to test it out in baby steps.

~~~~ { .scala }
package linkedData

import org.parboiled2._

class NTriples(val input: ParserInput) extends Parser {

  case class Triple(subj: String, pred: String, obj: String)

  def line = rule { capture(zeroOrMore(CharPredicate.Printable)) ~>
    (x => Triple(x,x,x)) }

  // comment ::= '#' (character - ( cr | lf ) )*  
  private def comment = ???

  // triple ::= subject ws+ predicate ws+ object ws* '.' ws*   
  private def triple = ???

  // subject ::= uriref | namedNode   
  private def subject = ???

  // predicate ::= uriref   
  private def predicate = ???

  // object ::= uriref | namedNode | literal   
  private def obj = ???

  // uriref ::= '<' absoluteURI '>'  
  private def uriref = ???

  // namedNode ::= '_:' name  
  private def namedNode = ???

  // literal ::= '"' string '"'   
  private def literal = ???

  // absoluteURI ::= ( character - ( '<' | '>' | space ) )+   
  private def absoluteUri = ???

  // We don't need end-of-line stuff, since we are processing one line at a time
  // eoln ::= cr | lf | cr lf  
  // cr ::= #xD /* US-ASCII carriage return - decimal 13 */  
  // lf ::= #xA /* US-ASCII linefeed - decimal 10 */   

  // ws ::= space | tab  
  private def ws = ???

  // space ::= #x20 /* US-ASCII space - decimal 32 */   
  private def space = ???

  // tab ::= #x9 /* US-ASCII horizontal tab - decimal 9 */  
  private def tab = ???

  // string ::= character* with escapes. Defined in section Strings  
  private def string = ???

  // name ::= [A-Za-z][A-Za-z0-9]*   
  private def name = ???

  // character ::= [#x20-#x7E] /* US-ASCII space to decimal 127 */ 
  private def character = ???
}
~~~~

The only slightly interesting thing to note here is the [parser action](https://github.com/sirthias/parboiled2#parser-actions) (`~>`) after the `capture` that constructs the output object we want.  What is happening here is that the `capture` is pushing an object (`string`) on the parser stack.  The parser action lambda pops this from the stack (since we are "asking" for a single object (`x`), that's all that will be given to our lambda).  The action, in this case, just uses the captured string to populate a `Triple` (which is implicitly pushed on the stack).

We haven't explicitly specified a type for `line`, but it is a `Rule1[Triple]` --- which pushes exactly one object of type `Triple` onto the stack.

Here’s a trivial driver to get us going:

~~~~ { .scala }
package linkedData

import scala.collection.immutable

object Main {
  def main(args: Array[String]) : Unit = {

    val ntDoc = 
    """

    # This is a comment
    <http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2000/01/rdf-schema#label> "Concept"@en .
    <http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2000/01/rdf-schema#isDefinedBy> <http://www.w3.org/2004/02/skos/core> .
    <http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2004/02/skos/core#definition> "An idea or notion; a unit of thought."@en .
    <http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <http://www.w3.org/2002/07/owl#Class> .
    _:BX2Db3de8bfX3A149861d9206X3AX2D7ffe <http://www.w3.org/1999/02/22-rdf-syntax-ns#first> <http://www.w3.org/2004/02/skos/core#Concept> .
    _:BX2Db3de8bfX3A149861d9206X3AX2D7ffe <http://www.w3.org/1999/02/22-rdf-syntax-ns#rest> _:BX2Db3de8bfX3A149861d9206X3AX2D7ffd .
    
    """

    val triples = ntDoc.lines map {line => new NTriples(line).line.run()}

    triples.foreach(println)
  }  
}
~~~~

The output of which is:

~~~~ { .bash }
Success(Triple(,,))
Success(Triple(<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2000/01/rdf-schema#label> "Concept"@en .,<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2000/01/rdf-schema#label> "Concept"@en .,<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2000/01/rdf-schema#label> "Concept"@en .))
Success(Triple(<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2000/01/rdf-schema#isDefinedBy> <http://www.w3.org/2004/02/skos/core> .,<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2000/01/rdf-schema#isDefinedBy> <http://www.w3.org/2004/02/skos/core> .,<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2000/01/rdf-schema#isDefinedBy> <http://www.w3.org/2004/02/skos/core> .))
Success(Triple(<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2004/02/skos/core#definition> "An idea or notion; a unit of thought."@en .,<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2004/02/skos/core#definition> "An idea or notion; a unit of thought."@en .,<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2004/02/skos/core#definition> "An idea or notion; a unit of thought."@en .))
Success(Triple(<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <http://www.w3.org/2002/07/owl#Class> .,<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <http://www.w3.org/2002/07/owl#Class> .,<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <http://www.w3.org/2002/07/owl#Class> .))
Success(Triple(_:BX2Db3de8bfX3A149861d9206X3AX2D7ffe <http://www.w3.org/1999/02/22-rdf-syntax-ns#first> <http://www.w3.org/2004/02/skos/core#Concept> .,_:BX2Db3de8bfX3A149861d9206X3AX2D7ffe <http://www.w3.org/1999/02/22-rdf-syntax-ns#first> <http://www.w3.org/2004/02/skos/core#Concept> .,_:BX2Db3de8bfX3A149861d9206X3AX2D7ffe <http://www.w3.org/1999/02/22-rdf-syntax-ns#first> <http://www.w3.org/2004/02/skos/core#Concept> .))
Success(Triple(_:BX2Db3de8bfX3A149861d9206X3AX2D7ffe <http://www.w3.org/1999/02/22-rdf-syntax-ns#rest> _:BX2Db3de8bfX3A149861d9206X3AX2D7ffd .,_:BX2Db3de8bfX3A149861d9206X3AX2D7ffe <http://www.w3.org/1999/02/22-rdf-syntax-ns#rest> _:BX2Db3de8bfX3A149861d9206X3AX2D7ffd .,_:BX2Db3de8bfX3A149861d9206X3AX2D7ffe <http://www.w3.org/1999/02/22-rdf-syntax-ns#rest> _:BX2Db3de8bfX3A149861d9206X3AX2D7ffd .))
Success(Triple(    ,    ,    ))
~~~~

Not exactly what we want, but it’s a start…

## Baby Step 1

We could filter out those blank lines from the input, but our parser should really deal with those, since they could come from the files that way.  We could also return an `Option` or an `Either`, but the return type of the `Parser` is already a `Success`.  Instead, let's define a `Line` type and derive the possible variants: `Blank` lines, `Comment` lines, and actual `Triples`.

~~~~ { .scala }
  trait Line       // A parsed line; can either be Blank, Comment, or Triple

  case class Blank() extends Line
  case class Comment(text: String) extends Line
  case class Triple(subj: Subject, pred: Predicate, obj: Object) extends Line

~~~~

Ah, notice that when it comes to defining a `Triple`, we actually need to have stronger types for `Subject`, `Predicate`, and `Object`.  Because these types have some rules as to what each are allowed to be, we can define some variants: `UriRef` and `NamedNode`.

~~~~ { .scala }
  trait Subject    // A triple's subject; can either be a uriref or namednode
  trait Predicate  // A triple's predicate; can either be a uriref or namednode
  trait Object     // A triple's object; will always be a uriref

  case class UriRef(uri: String) extends Subject with Object with Predicate
  case class NamedNode(name: String) extends Subject with Predicate
~~~~

Now that we have some richer types to work with, we need to expand our parser rules.  `Line` really can either be blank or non-blank.  If it is non-blank, then it is either a `Comment` or a `Triple`.

~~~~ { .scala }
  // line ::= ws* (comment | triple) ? eoln  
  def line: Rule1[Line] = rule { blank | nonBlank }
  private def blank: Rule1[Line] = rule { zeroOrMore(ws) ~ EOI ~ push(Blank()) }
  private def nonBlank: Rule1[Line] = rule { (comment | triple) ~ EOI }
~~~~

Note that I am now using explicit typing.  This isn't strictly necessary, but it helps to make clear what is flowing between the various parsers.

For a blank line, we have a rule which is looking for 0+ whitespace characters followed by the end-of-input (EOI), at which point we push a new `Blank` object onto the stack.

For a non-blank line, we have a rule that expects either a `Comment` or a `Triple` followed by EOI.

### Whitespace
One of the often cited downsides of using PEGs is the need to deal with whitespace explicitly, instead of having a lexer deal with it.  Let's first define what we mean by whitespace and fill in the BNF definitions we are given:

~~~~ { .scala }
  // ws ::= space | tab  
  private def ws = rule { space | tab }  

  // space ::= #x20 /* US-ASCII space - decimal 32 */   
  private def space = CharPredicate(' ')

  // tab ::= #x9 /* US-ASCII horizontal tab - decimal 9 */  
  private def tab = CharPredicate('\t')
~~~~

Whitespace, for us, is just a space or tab character.  `CharPredicates` are predefined for a number of different character groups, such as digits, alpha, etc.  They are rules of type Rule0, so they just consume their matching input and push nothing onto the stack.  In this case, we create a custom predicate for each of the characters we are interested in (space and tab).

### Comments
A comment is simply a line that begins with '#'.  We want to *capture* the comment text and return it.  In PB2, capturing places a string onto the stack.  Here, we use the predefined character predicate `Printable`:

~~~~ { .scala }
  // comment ::= '#' (character - ( cr | lf ) )*  
  private def comment: Rule1[Comment] = rule {
    '#' ~ zeroOrMore(ws) ~ capture(zeroOrMore(CharPredicate.Printable)) ~> 
      (Comment(_))
  }
~~~~

### Triples
The last top-level object we are interested in is the actual `Triple`.  A triple is a `Subject`, `Predicate`, `Object` followed by a '.', taking into account whitespace.  But for now, in the spirit of baby steps, let's just stub out the `Subject` with a simple line and ignore the other two types.

~~~~ { .scala }
  // triple ::= subject ws+ predicate ws+ object ws* '.' ws*   
  private def triple: Rule1[Triple] = rule {
    zeroOrMore(ws) ~ subject ~>
      ((s : Subject) => Triple(s,UriRef(""),UriRef("")))
  }

  // subject ::= uriref | namedNode   
  private def subject: Rule1[Subject] = rule {
    capture(zeroOrMore(CharPredicate.Printable)) ~> (c => UriRef(c))
  }
~~~~

The current output doesn't look much different, though we've made good progress:

~~~~ { .bash }
Success(Blank())
Success(Blank())
Success(Comment(This is a comment))
Success(Triple(UriRef(<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2000/01/rdf-schema#label> "Concept"@en .),UriRef(),UriRef()))
Success(Triple(UriRef(<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2000/01/rdf-schema#isDefinedBy> <http://www.w3.org/2004/02/skos/core> .),UriRef(),UriRef()))
Success(Triple(UriRef(<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2004/02/skos/core#definition> "An idea or notion; a unit of thought."@en .),UriRef(),UriRef()))
Success(Triple(UriRef(<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <http://www.w3.org/2002/07/owl#Class> .),UriRef(),UriRef()))
Success(Triple(UriRef(_:BX2Db3de8bfX3A149861d9206X3AX2D7ffe <http://www.w3.org/1999/02/22-rdf-syntax-ns#first> <http://www.w3.org/2004/02/skos/core#Concept> .),UriRef(),UriRef()))
Success(Triple(UriRef(_:BX2Db3de8bfX3A149861d9206X3AX2D7ffe <http://www.w3.org/1999/02/22-rdf-syntax-ns#rest> _:BX2Db3de8bfX3A149861d9206X3AX2D7ffd .),UriRef(),UriRef()))
Success(Blank())
~~~~

## Baby Step 2