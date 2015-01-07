# Overview

Our goal is to use the [PEG-based parser](http://en.wikipedia.org/wiki/Parsing_expression_grammar) [Parboild2](https://github.com/sirthias/parboiled2) in order to parse [N-Triples](http://www.w3.org/2001/sw/RDFCore/ntriples) documents as part of [Maana’s](http://www.crunchbase.com/organization/maana) support for [Linked Data](http://linkeddata.org).

Our test project for this article can be found [here](https://github.com/witt3rd/linked-data). We won’t bother going into the setup or basic concepts of PEGs or PB2, since they are all covered well enough on the PB2 Github page (e.g., you should have a basic grasp of the parser stack and RuleN[T] mechanism). Instead, we are going to just dive right in building our grammar and parsing some sample files by incrementally building up the solution through a series of baby steps.  (Yes, this approach would have been well-served by using TDD.)

My motivation for this (and the approach taken here) is that there are some nuances in PB2 that aren't immediately obvious from the documentation and the examples are either too simplistic or too elaborate.  And the examples are always complete --- you just see the end result without getting a sense for the reasoning behind various decisions.

# N-Triples

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

# Baby Step 0: Scaffold

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

    triples foreach println
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

# Baby Step 1: Line Types

We could filter out those blank lines from the input, but our parser should really deal with those, since they could come from the files that way.  We could also return an `Option` or an `Either`, but the return type of the `Parser` is already a `Try[T]`.  Instead, let's define a `Line` type and derive the possible variants: `Blank` lines, `Comment` lines, and actual `Triples`.

~~~~ { .scala }
  trait Line       // A parsed line; can either be Blank, Comment, or Triple

  case class Blank() extends Line
  case class Comment(text: String) extends Line
  case class Triple(subj: Subject, pred: Predicate, obj: Object) extends Line

~~~~

Ah, notice that when it comes to defining a `Triple`, we actually need to have stronger types for `Subject`, `Predicate`, and `Object`.  Because these types have some rules as to what each are allowed to be, we can define some variants: `UriRef`, `NamedNode`, and `Literal`.

~~~~ { .scala }
  trait Subject    // A triple's subject; can either be a uriref or namednode
  trait Predicate  // A triple's predicate; can either be a uriref or namednode
  trait Object     // A triple's object; will always be a uriref or namednode or literal

  case class UriRef(uri: String) extends Subject with Object with Predicate
  case class NamedNode(name: String) extends Subject with Object with Predicate
  case class Literal(value: String) extends Object
~~~~

Now that we have some richer types to work with, we need to expand our parser rules.  `Line` really can either be blank or non-blank.  If it is non-blank, then it is either a `Comment` or a `Triple`.

~~~~ { .scala }
  // line ::= ws* (comment | triple) ? eoln  
  def line: Rule1[Line] = rule { blank | nonBlank }
  private def blank: Rule1[Line] = rule { zeroOrMore(ws) ~ EOI ~ push(Blank()) }
  private def nonBlank: Rule1[Line] = rule { (comment | triple) ~ EOI }
~~~~

Note that we are now using explicit typing.  This isn't strictly necessary, but it helps to make clear what is flowing between the various parsers.

For a blank line, we have a rule which is looking for 0+ whitespace characters followed by the end-of-input (EOI), at which point we push a new `Blank` object onto the stack.

For a non-blank line, we have a rule that expects either a `Comment` or a `Triple` followed by EOI.

## Whitespace
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

## Comments
A comment is simply a line that begins with '#'.  We want to *capture* the comment text and return it.  In PB2, capturing places a string onto the stack.  Here, we use the predefined character predicate `Printable`:

~~~~ { .scala }
  // comment ::= '#' (character - ( cr | lf ) )*  
  private def comment: Rule1[Comment] = rule {
    '#' ~ zeroOrMore(ws) ~ capture(zeroOrMore(CharPredicate.Printable)) ~> 
      (Comment(_))
  }
~~~~

## Triples
The last top-level object we are interested in is the actual `Triple`.  A triple is a `Subject`, `Predicate`, `Object` followed by a '.', taking into account whitespace.  But for now, in the spirit of baby steps, let's just stub out the `Subject` with a simple line and ignore the other two types.

~~~~ { .scala }
  // triple ::= subject ws+ predicate ws+ object ws* '.' ws*   
  private def triple: Rule1[Triple] = rule {
    zeroOrMore(ws) ~ subject ~>
      ((s : Subject) => Triple(s, UriRef(""), UriRef("")))
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

# Baby Step 2: UriRefs
Now we need to actually start parsing a real line.  The first place to start is parsing `urirefs`, which are standard URLs wrapped in '<' and '>'.  Let's start by revising our `triple` and `subject` rules as follows:

~~~~ { .scala }
  private def triple: Rule1[Triple] = rule {
    zeroOrMore(ws) ~ subject ~ zeroOrMore(ANY) ~>
      ((s : Subject) => Triple(s, UriRef(""), UriRef("")))
  }

  // subject ::= uriref | namedNode   
  private def subject: Rule1[Subject] = rule {
    '<' ~ capture(zeroOrMore(!'>' ~ ANY)) ~> (UriRef(_))
  }
~~~~

Notice that we've added a `zeroOrMore(ANY)` to our `triple` rule.  This allows us to match the subject, which is just one component of a triple, followed by anything else until EOI.

For `subject`, we are specifically looking to capture the content between '<' and '>', which makes the subject a `UriRef` (we'll fix this next).

The result of this change yields:

~~~~ { .bash }
Success(Blank())
Success(Blank())
Success(Comment(This is a comment))
Success(Triple(UriRef(http://www.w3.org/2004/02/skos/core#Concept),UriRef(),UriRef()))
Success(Triple(UriRef(http://www.w3.org/2004/02/skos/core#Concept),UriRef(),UriRef()))
Success(Triple(UriRef(http://www.w3.org/2004/02/skos/core#Concept),UriRef(),UriRef()))
Success(Triple(UriRef(http://www.w3.org/2004/02/skos/core#Concept),UriRef(),UriRef()))
Failure(org.parboiled2.ParseError)
Failure(org.parboiled2.ParseError)
Success(Blank())
~~~~

Note that our `Triple` now contains one valid `UriRef` and that the last two lines have failed to parse (recall: they are `NamedNodes`).

# Baby Step 3: NamedNodes and UriRefs
Let's now add support for `NamedNodes` and allow `Subject` to be either a `NamedNode` or a `UriRef`.

~~~~ { .scala }
  // subject ::= uriref | namedNode   
  private def subject: Rule1[Subject] = rule {
    uriref | namedNode
  }

  // uriref ::= '<' absoluteURI '>'  
  private def uriref: Rule1[UriRef] = rule {
    '<' ~ capture(zeroOrMore(!'>' ~ ANY)) ~ '>' ~> (UriRef(_))    
  }

  // namedNode ::= '_:' name  
  private def namedNode: Rule1[NamedNode] = rule {
    '_' ~ ':' ~ capture(name) ~> (NamedNode(_))    
  }

  // name ::= [A-Za-z][A-Za-z0-9]*   
  private def name = rule {
    CharPredicate.Alpha ~ zeroOrMore(CharPredicate.AlphaNum)
  }
~~~~

Here we have moved the previous parser logic from `subject` into `uriref` and replaced it with either `uriref` or `namedNode` and we've added a parser definition for `namedNode`, which relies on the `name` parser.  This is all quite straightforward except for perhaps the definition of `uriref` which is looking for zero or more non-'>' characters *followed by* a '>' character.

This step produces the following output:

~~~~ { .bash }
Success(Blank())
Success(Blank())
Success(Comment(This is a comment))
Success(Triple(UriRef(http://www.w3.org/2004/02/skos/core#Concept),UriRef(),UriRef()))
Success(Triple(UriRef(http://www.w3.org/2004/02/skos/core#Concept),UriRef(),UriRef()))
Success(Triple(UriRef(http://www.w3.org/2004/02/skos/core#Concept),UriRef(),UriRef()))
Success(Triple(UriRef(http://www.w3.org/2004/02/skos/core#Concept),UriRef(),UriRef()))
Success(Triple(NamedNode(BX2Db3de8bfX3A149861d9206X3AX2D7ffe),UriRef(),UriRef()))
Success(Triple(NamedNode(BX2Db3de8bfX3A149861d9206X3AX2D7ffe),UriRef(),UriRef()))
Success(Blank())
~~~~

Note that we now successfully get `NamedNodes` as the `Subject` for our last two `Triples`.

# Baby Step 4: Objects and Predicates
Now we just need to add back our `Predicate` and `Object` parsers with the appropriate definitions:

~~~~ { .scala }
  // triple ::= subject ws+ predicate ws+ object ws* '.' ws*   
  private def triple: Rule1[Triple] = rule { 
    zeroOrMore(ws) ~ subject ~ 
    zeroOrMore(ws) ~ predicate ~ 
    oneOrMore(ws)  ~ obj ~ 
    zeroOrMore(ws) ~ '.' ~ 
    zeroOrMore(ws) ~>
      ((s: Subject, p: Predicate, o: Object) => Triple(s, p, o))
  }

  // predicate ::= uriref   
  private def predicate: Rule1[Predicate] = rule {
    uriref
  }

  // object ::= uriref | namedNode | literal   
  private def obj: Rule1[Object] = rule {
    uriref | namedNode
  }
~~~~

Again, nothing surprising here, just an expansion of our `Triple` parser to include `Predicate` and `Object` parses and the addition of definitions for `predicate` and `obj` parsers.  This now yields:

~~~~ { .bash }
Success(Blank())
Success(Blank())
Success(Comment(This is a comment))
Failure(org.parboiled2.ParseError)
Success(Triple(UriRef(http://www.w3.org/2004/02/skos/core#Concept),UriRef(http://www.w3.org/2000/01/rdf-schema#isDefinedBy),UriRef(http://www.w3.org/2004/02/skos/core)))
Failure(org.parboiled2.ParseError)
Success(Triple(UriRef(http://www.w3.org/2004/02/skos/core#Concept),UriRef(http://www.w3.org/1999/02/22-rdf-syntax-ns#type),UriRef(http://www.w3.org/2002/07/owl#Class)))
Success(Triple(NamedNode(BX2Db3de8bfX3A149861d9206X3AX2D7ffe),UriRef(http://www.w3.org/1999/02/22-rdf-syntax-ns#first),UriRef(http://www.w3.org/2004/02/skos/core#Concept)))
Success(Triple(NamedNode(BX2Db3de8bfX3A149861d9206X3AX2D7ffe),UriRef(http://www.w3.org/1999/02/22-rdf-syntax-ns#rest),NamedNode(BX2Db3de8bfX3A149861d9206X3AX2D7ffd)))
Success(Blank())
~~~~

We can now parse almost all the lines --- except we don't have a full definitions of `Object` since we don't yet support `Literals`.

# Baby Step 5: Literals
Going by the spec, `Literals` are just strings.  However, the reality is that these strings often have the language specifiers added.  This complicates parsing a little bit.

~~~~ { .scala }
  // object ::= uriref | namedNode | literal   
  private def obj: Rule1[Object] = rule {
    uriref | namedNode | literal
  }

  // literal ::= '"' string '"'   
  private def literal: Rule1[Literal] = rule {
    '\"' ~ capture(zeroOrMore(!'\"' ~ ANY)) ~ '\"' ~ zeroOrMore(!'.' ~ ANY) ~> (Literal(_))
  }
~~~~

First, we extend the definition of `obj` to include `literal`.  Next, we define the rule for `literals` as being a string (any characters between double quotes) followed by any non-'.' characters (since '.' signifies the end of the line).  Of course, this fails to account for embedded escapes, etc., but we can add these things later.  For now, we just want to get the majority of cases working.

Finally, the output for our test input is:

~~~~ { .bash }
Success(Blank())
Success(Blank())
Success(Comment(This is a comment))
Success(Triple(UriRef(http://www.w3.org/2004/02/skos/core#Concept),UriRef(http://www.w3.org/2000/01/rdf-schema#label),Literal(Concept)))
Success(Triple(UriRef(http://www.w3.org/2004/02/skos/core#Concept),UriRef(http://www.w3.org/2000/01/rdf-schema#isDefinedBy),UriRef(http://www.w3.org/2004/02/skos/core)))
Success(Triple(UriRef(http://www.w3.org/2004/02/skos/core#Concept),UriRef(http://www.w3.org/2004/02/skos/core#definition),Literal(An idea or notion; a unit of thought.)))
Success(Triple(UriRef(http://www.w3.org/2004/02/skos/core#Concept),UriRef(http://www.w3.org/1999/02/22-rdf-syntax-ns#type),UriRef(http://www.w3.org/2002/07/owl#Class)))
Success(Triple(NamedNode(BX2Db3de8bfX3A149861d9206X3AX2D7ffe),UriRef(http://www.w3.org/1999/02/22-rdf-syntax-ns#first),UriRef(http://www.w3.org/2004/02/skos/core#Concept)))
Success(Triple(NamedNode(BX2Db3de8bfX3A149861d9206X3AX2D7ffe),UriRef(http://www.w3.org/1999/02/22-rdf-syntax-ns#rest),NamedNode(BX2Db3de8bfX3A149861d9206X3AX2D7ffd)))
Success(Blank())
~~~~

# Error Handling
So far, our parse errors have come from our lack of support for the format.  Now that the parser works, let's see how to deal with the results, including errors.

## Better Driver
Let's first make our driver a bit more sophisticated and refactor things a bit.  What we want to do is retain our trivial test, since it is very easy to understand what's going on, but also now take a filename specifying an .nt file and process it line-by-line.

Additionally, let's refactor things a bit by having a common parse method that accepts a line iterator and performs the parse and prints out the results.

Lastly, let's introduce an error into our test data.

~~~~ { .scala }
package linkedData

import scala.collection.immutable
import scala.io.Source

object Main {
  def main(args: Array[String]) : Unit = {

    unitTest()
    //return

    if (args.size < 1) {
      println("missing .nt file")
      return
    }

    val Array(ntFile) = args

    val lines = Source.fromFile(ntFile).getLines()
    parse(lines)
  }

  private def parse(lines: Iterator[String]) : Unit = {
    val triples = lines map {l => new NTriples(l).line.run()}
    triples foreach println
  }

  private def unitTest() : Unit = {

    val ntDoc = 
    """

# This is a comment
<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2000/01/rdf-schema#label> "Concept"@en .
<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2000/01/rdf-schema#isDefinedBy> <http://www.w3.org/2004/02/skos/core> .
<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/2004/02/skos/core#definition> "An idea or notion; a unit of thought."@en .
<http://www.w3.org/2004/02/skos/core#Concept> <http://www.w3.org/1999/02/22-rdf-syntax-ns#type> <http://www.w3.org/2002/07/owl#Class> .
_:BX2Db3de8bfX3A149861d9206X3AX2D7ffe <http://www.w3.org/1999/02/22-rdf-syntax-ns#first> <http://www.w3.org/2004/02/skos/core#Concept> .
!!! ERROR ERROR ERROR ERROR ERROR
_:BX2Db3de8bfX3A149861d9206X3AX2D7ffe <http://www.w3.org/1999/02/22-rdf-syntax-ns#rest> _:BX2Db3de8bfX3A149861d9206X3AX2D7ffd .
    """

    parse(ntDoc.lines)
  }
}
~~~~

The output is the same as previously except for the newly introduced parse error.

## Companion Object
Since the next thing we wish to do is access the results, whether success or failure, we need to refactor our parser module by extracting our result object definitions from the class instance into a companion object.

~~~~ { .scala }
object NTriples {
  // Main parsed types
  trait Line       // A parsed line; can either be Blank, Comment, or Triple
  trait Subject    // A triple's subject; can either be a uriref or namednode
  trait Predicate  // A triple's predicate; can either be a uriref or namednode
  trait Object     // A triple's object; will always be a uriref or or namednode or literal

  case class UriRef(uri: String) extends Subject with Object with Predicate
  case class NamedNode(name: String) extends Subject with Object with Predicate
  case class Literal(value: String) extends Object

  case class Blank() extends Line
  case class Comment(text: String) extends Line
  case class Triple(subj: Subject, pred: Predicate, obj: Object) extends Line
}

class NTriples(val input: ParserInput) extends Parser {

  import NTriples._

  ...
~~~~

With this change, the caller can import these definitions and use them to handle the results.

## Matching Results
The parser returns either `Success` or `Failure` (part of Scala's `util` library).  If it is `Success`, it will be one of our objects (`Blank`, `Comment`, or `Triple`).  If it is a `Failure`, it will either be a `ParseError` or other (unexpected) error.

There is a problem in our current formulation: we iterate through the lines and create a new parser and perform the parse, storing the result in a new collection.  But when a `ParseError` occurs, we need the actual parser *instance* in order to format it properly.

To deal with this situation, let's construct the parser, parse the line, and if there is an error, capture the formatted error *in situ*, returning it as a `Failure`, otherwise passing the original result directly through.  This is just translating the error inline.

Let's add this to our companion object as a helper function.

~~~~ { .scala }
import scala.util.{ Try, Success, Failure }

object NTriples {

  ...

  def parse(lines: Iterator[String]) : Iterator[Try[Line]] = {
    lines map {l => 
      val p = new NTriples(l)
      p.line.run() match {
        case Failure(e: ParseError) => Failure(new Exception(p.formatError(e)))
        case x                      => x
      }
    }    
  }
}
~~~~

## Updated Driver, Better Printing
We can now use the helper function and the deal with the results.

~~~~ { .scala }
import scala.util.{Success, Failure}
import org.parboiled2.{ParseError}

...

  private def parse(lines: Iterator[String]) : Unit = {
    import NTriples._

    val triples = NTriples.parse(lines)

    triples foreach {_ match {
        case Success(t@Triple(s,p,o)) => println(t)
        case Success(_)               =>
        case Failure(e: Throwable)    => println("Expression is not valid: " + e)
      }}
  }
~~~~

We now just print out valid `Triples` and any parse errors.  Here, then, is the final output using our little test data:

~~~~ { .bash }
Triple(UriRef(http://www.w3.org/2004/02/skos/core#Concept),UriRef(http://www.w3.org/2000/01/rdf-schema#label),Literal(Concept))
Triple(UriRef(http://www.w3.org/2004/02/skos/core#Concept),UriRef(http://www.w3.org/2000/01/rdf-schema#isDefinedBy),UriRef(http://www.w3.org/2004/02/skos/core))
Triple(UriRef(http://www.w3.org/2004/02/skos/core#Concept),UriRef(http://www.w3.org/2004/02/skos/core#definition),Literal(An idea or notion; a unit of thought.))
Triple(UriRef(http://www.w3.org/2004/02/skos/core#Concept),UriRef(http://www.w3.org/1999/02/22-rdf-syntax-ns#type),UriRef(http://www.w3.org/2002/07/owl#Class))
Triple(NamedNode(BX2Db3de8bfX3A149861d9206X3AX2D7ffe),UriRef(http://www.w3.org/1999/02/22-rdf-syntax-ns#first),UriRef(http://www.w3.org/2004/02/skos/core#Concept))
Expression is not valid: java.lang.Exception: Invalid input '!', expected space, tab, 'EOI', '#', '<' or '_' (line 1, column 1):
!!! ERROR ERROR ERROR ERROR ERROR
^
Triple(NamedNode(BX2Db3de8bfX3A149861d9206X3AX2D7ffe),UriRef(http://www.w3.org/1999/02/22-rdf-syntax-ns#rest),NamedNode(BX2Db3de8bfX3A149861d9206X3AX2D7ffd))
~~~~

# Conclusion
More checking (such as URLs are really URLs) needs to be done, but this is the basic gist of it.

For the fully working version with more tests and driver, check it out on [Github](https://github.com/witt3rd/linked-data).