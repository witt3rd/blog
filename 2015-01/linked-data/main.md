# Overview

Our goal is to use the [PEG-based parser](http://en.wikipedia.org/wiki/Parsing_expression_grammar) [Parboild2](https://github.com/sirthias/parboiled2) in order to parse [N-Triples](http://www.w3.org/2001/sw/RDFCore/ntriples) documents as part of [Maana’s](http://maana.io) support for [Linked Data](http://linkeddata.org).

Our test project for this article can be found [here](https://github.com/witt3rd/linked-data). I won’t bother going into the setup or basic concepts of PEGs or PB2, since they are all covered well enough on the PB2 Github page. Instead, we are going to just dive right in building our grammar and parsing some sample files. In the next part, we’ll expand beyond the simple N-Triples format to the slightly more complex [RDF/Turtle](http://www.w3.org/TeamSubmission/turtle) format.

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

For now, we’ll just parse a line (string) and output a triple consisting of the string repeated for subject, predicate, object, just to keep things simple and to test it out in baby steps.

~~~~ { .scala }
package linkedData

import org.parboiled2._

class NTriples(val input: ParserInput) extends Parser {

  case class Triple(subj: String, pred: String, obj: String)

  def line = rule { capture(zeroOrMore(CharPredicate.Printable)) ~> (x => Triple(x,x,x)) }

  def comment = ???
  def triple = ???
  def subject = ???
  def predicate = ???
  def obj = ???
  def uriref = ???
  def namedNode = ???
  def literal = ???
  def absoluteUri = ???
}
~~~~

The only slightly interesting thing to note here is the [parser action](https://github.com/sirthias/parboiled2#parser-actions)) (`~>`) after the `capture` that constructs the output object we want.

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
Success(Triple([http://www.w3.org/2004/02/skos/core#Concept](http://www.w3.org/2004/02/skos/core#Concept) [http://www.w3.org/2000/01/rdf-schema#label](http://www.w3.org/2000/01/rdf-schema#label) “Concept”<span class="citation">(<span class="citeproc-not-found" data-reference-id="en">**???**</span>)</span> .,[http://www.w3.org/2004/02/skos/core#Concept](http://www.w3.org/2004/02/skos/core#Concept) [http://www.w3.org/2000/01/rdf-schema#label](http://www.w3.org/2000/01/rdf-schema#label) “Concept”<span class="citation">(<span class="citeproc-not-found" data-reference-id="en">**???**</span>)</span> .,[http://www.w3.org/2004/02/skos/core#Concept](http://www.w3.org/2004/02/skos/core#Concept) [http://www.w3.org/2000/01/rdf-schema#label](http://www.w3.org/2000/01/rdf-schema#label) “Concept”<span class="citation">(<span class="citeproc-not-found" data-reference-id="en">**???**</span>)</span> .))
Success(Triple([http://www.w3.org/2004/02/skos/core#Concept](http://www.w3.org/2004/02/skos/core#Concept) [http://www.w3.org/2000/01/rdf-schema#isDefinedBy](http://www.w3.org/2000/01/rdf-schema#isDefinedBy) [http://www.w3.org/2004/02/skos/core](http://www.w3.org/2004/02/skos/core) .,[http://www.w3.org/2004/02/skos/core#Concept](http://www.w3.org/2004/02/skos/core#Concept) [http://www.w3.org/2000/01/rdf-schema#isDefinedBy](http://www.w3.org/2000/01/rdf-schema#isDefinedBy) [http://www.w3.org/2004/02/skos/core](http://www.w3.org/2004/02/skos/core) .,[http://www.w3.org/2004/02/skos/core#Concept](http://www.w3.org/2004/02/skos/core#Concept) [http://www.w3.org/2000/01/rdf-schema#isDefinedBy](http://www.w3.org/2000/01/rdf-schema#isDefinedBy) [http://www.w3.org/2004/02/skos/core](http://www.w3.org/2004/02/skos/core) .))
Success(Triple([http://www.w3.org/2004/02/skos/core#Concept](http://www.w3.org/2004/02/skos/core#Concept) [http://www.w3.org/2004/02/skos/core#definition](http://www.w3.org/2004/02/skos/core#definition) “An idea or notion; a unit of thought.”<span class="citation">(<span class="citeproc-not-found" data-reference-id="en">**???**</span>)</span> .,[http://www.w3.org/2004/02/skos/core#Concept](http://www.w3.org/2004/02/skos/core#Concept) [http://www.w3.org/2004/02/skos/core#definition](http://www.w3.org/2004/02/skos/core#definition) “An idea or notion; a unit of thought.”<span class="citation">(<span class="citeproc-not-found" data-reference-id="en">**???**</span>)</span> .,[http://www.w3.org/2004/02/skos/core#Concept](http://www.w3.org/2004/02/skos/core#Concept) [http://www.w3.org/2004/02/skos/core#definition](http://www.w3.org/2004/02/skos/core#definition) “An idea or notion; a unit of thought.”<span class="citation">(<span class="citeproc-not-found" data-reference-id="en">**???**</span>)</span> .))
Success(Triple([http://www.w3.org/2004/02/skos/core#Concept](http://www.w3.org/2004/02/skos/core#Concept) [http://www.w3.org/1999/02/22-rdf-syntax-ns#type](http://www.w3.org/1999/02/22-rdf-syntax-ns#type) [http://www.w3.org/2002/07/owl#Class](http://www.w3.org/2002/07/owl#Class) .,[http://www.w3.org/2004/02/skos/core#Concept](http://www.w3.org/2004/02/skos/core#Concept) [http://www.w3.org/1999/02/22-rdf-syntax-ns#type](http://www.w3.org/1999/02/22-rdf-syntax-ns#type) [http://www.w3.org/2002/07/owl#Class](http://www.w3.org/2002/07/owl#Class) .,[http://www.w3.org/2004/02/skos/core#Concept](http://www.w3.org/2004/02/skos/core#Concept) [http://www.w3.org/1999/02/22-rdf-syntax-ns#type](http://www.w3.org/1999/02/22-rdf-syntax-ns#type) [http://www.w3.org/2002/07/owl#Class](http://www.w3.org/2002/07/owl#Class) .))
Success(Triple(_:BX2Db3de8bfX3A149861d9206X3AX2D7ffe [http://www.w3.org/1999/02/22-rdf-syntax-ns#first](http://www.w3.org/1999/02/22-rdf-syntax-ns#first) [http://www.w3.org/2004/02/skos/core#Concept](http://www.w3.org/2004/02/skos/core#Concept) .,_:BX2Db3de8bfX3A149861d9206X3AX2D7ffe [http://www.w3.org/1999/02/22-rdf-syntax-ns#first](http://www.w3.org/1999/02/22-rdf-syntax-ns#first) [http://www.w3.org/2004/02/skos/core#Concept](http://www.w3.org/2004/02/skos/core#Concept) .,_:BX2Db3de8bfX3A149861d9206X3AX2D7ffe [http://www.w3.org/1999/02/22-rdf-syntax-ns#first](http://www.w3.org/1999/02/22-rdf-syntax-ns#first) [http://www.w3.org/2004/02/skos/core#Concept](http://www.w3.org/2004/02/skos/core#Concept) .))
Success(Triple(_:BX2Db3de8bfX3A149861d9206X3AX2D7ffe [http://www.w3.org/1999/02/22-rdf-syntax-ns#rest](http://www.w3.org/1999/02/22-rdf-syntax-ns#rest) _:BX2Db3de8bfX3A149861d9206X3AX2D7ffd .,_:BX2Db3de8bfX3A149861d9206X3AX2D7ffe [http://www.w3.org/1999/02/22-rdf-syntax-ns#rest](http://www.w3.org/1999/02/22-rdf-syntax-ns#rest) _:BX2Db3de8bfX3A149861d9206X3AX2D7ffd .,_:BX2Db3de8bfX3A149861d9206X3AX2D7ffe [http://www.w3.org/1999/02/22-rdf-syntax-ns#rest](http://www.w3.org/1999/02/22-rdf-syntax-ns#rest) _:BX2Db3de8bfX3A149861d9206X3AX2D7ffd .))
Success(Triple( , , ))
~~~~

Not exactly what we want, but it’s a start…

## Baby Step 1

We could filter out those blank lines from the input, but our parser should really deal with those, since they could come from the files that way. To do this, let’s return an `Option` instead of a `Triple`, since it isn’t an error. Also, we don’t want to return comments, so those will result in `None`, too.