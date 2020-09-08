---
title: Scala Logics and Codecs
layout: post
---

Exercise solutions for `Scala 99 Problems, Part III Logics and Codecs"

## 46. Truth tables for logical expressions.
Define functions and, or, nand, nor, xor, impl, and equ (for logical equivalence) which return true or false according to the result of their respective operations; e.g. and(A, B) is true if and only if both A and B are true.
```
scala> and(true, true)
res0: Boolean = true

scala> xor(true. true)
res1: Boolean = false
```
A logical expression in two variables can then be written as an function of two variables, e.g: (a: Boolean, b: Boolean) => and(or(a, b), nand(a, b))

Now, write a function called table2 which prints the truth table of a given logical expression in two variables.
```
scala> table2((a: Boolean, b: Boolean) => and(a, or(a, b)))
A     B     result
true  true  true
true  false true
false true  false
false false false
```


```scala
  def and(a: Boolean, b: => Boolean): Boolean = if (a) b else false

  def or(a: Boolean, b: => Boolean): Boolean = if (a) true else b

  def not(a: Boolean): Boolean = if (a) false else true

  def equ(a: Boolean, b: Boolean): Boolean = or(and(a, b), and(not(a), not(b)))

  def xor(a: Boolean, b: Boolean): Boolean = not(equ(a,b))

  def impl(a: Boolean, b: => Boolean): Boolean = if (a) b else true

  def nand(a: Boolean, b: Boolean): Boolean = not(and(a, b))

  val f: (Boolean, Boolean) => Boolean = (a, b) => and(a, xor(a, b))

  def table2(f: (Boolean, Boolean) => Boolean): List[(Boolean, Boolean, Boolean)] = {
    List(true, false).flatMap(a => List(true, false).map(b => (a, b, f(a, b))))
  }
  table2((a: Boolean, b: Boolean) => and(a, or(a, b)))
```

## 47. Truth tables for logical expressions (2).
Continue problem P46 by redefining and, or, etc as operators. (i.e. make them methods of a new class with an implicit conversion from Boolean.) not will have to be left as a object method.

```
scala> table2((a: Boolean, b: Boolean) => a and (a or not(b)))
A     B     result
true  true  true
true  false true
false true  false
false false false
```


```scala
object P47 {
  implicit class BooleanWrapper(a: Boolean) {
    def and(b: => Boolean): Boolean  = if (a) b else false
    def or(b: => Boolean): Boolean   = if (a) true else b
    def nand(b: => Boolean): Boolean = if (a) !b else true
    def nor(b: => Boolean): Boolean  = if (a) false else !b
    def xor(b: Boolean): Boolean     = a != b
    def impl(b: => Boolean): Boolean = if (a) b else true
    def equ(b: Boolean): Boolean     = a == b
  }
}

import P47._
table2((a: Boolean, b: Boolean) => a and (a or not(b)))
```

## 48. Truth tables for logical expressions (3).
Omitted for now.

## 49. Gray code.
An n-bit Gray code is a sequence of n-bit strings constructed according to certain rules. For example,
```
n = 1: C(1) = ("0", "1").
n = 2: C(2) = ("00", "01", "11", "10").
n = 3: C(3) = ("000", "001", "011", "010", "110", "111", "101", "100").
```
Find out the construction rules and write a function to generate Gray codes.
```
scala> gray(3)
res0 List[String] = List(000, 001, 011, 010, 110, 111, 101, 100)
See if you can use memoization to make the function more efficient.
```


```scala
def gray(n: Int): List[String] = {
  if (n == 1) List("0", "1")
  else gray(n - 1).map("0" + _) ++ gray(n - 1).reverse.map("1" + _)
}

gray(3)
gray(4)
```

## 50. Huffman code.
First of all, consult a good book on discrete mathematics or algorithms for a detailed description of Huffman codes!
We suppose a set of symbols with their frequencies, given as a list of (S, F) Tuples. E.g. `(("a", 45), ("b", 13), ("c", 12), ("d", 16), ("e", 9), ("f", 5))`. Our objective is to construct a list of `(S, C)` Tuples, where C is the Huffman code word for the symbol S.

```
scala> huffman(List(("a", 45), ("b", 13), ("c", 12), ("d", 16), ("e", 9), ("f", 5)))
res0: List[String, String] = List((a,0), (b,101), (c,100), (d,111), (e,1101), (f,1100))
```

Algorithm implementation adapted from `Algorithms, 4th Edidtion, R. Sedgewick`

- build an encoding trie
- write the trie (encoded as a bitstream) for use in expansion (not required here for this problem)
- use the trie to encode the bytestream as a bitstream


```scala
 import scala.collection.mutable

  def huffman(input: List[(Char, Int)]) : mutable.Map[Char, String] = {
    val root = buildTrie(input)
    val map = mutable.Map[Char, String]().withDefaultValue("")
    def buildCode(trie: Node, s: String):Unit = {
      if (trie.isLeaf) map.put(trie.c, s)
      if (trie.left != null) buildCode(trie.left, s + "0")
      if (trie.right != null) buildCode(trie.right, s + "1")
    }
    buildCode(root, "")
    map
  }


  case class Node(c: Char, freq: Int, left: Node, right: Node) {
    def isLeaf: Boolean = left == null && right == null
  }

  object NodeOrdering extends Ordering[Node] {
    override def compare(x: Node, y: Node): Int = y.freq compare x.freq
  }

  def buildTrie(input: List[(Char, Int)]): Node = {
    // min heap
    val pq = mutable.PriorityQueue()(NodeOrdering)
    input.foreach(t => pq.enqueue(Node(t._1, t._2, null, null)))
    while(pq.size > 1) {
      val left = pq.dequeue()
      val right = pq.dequeue()
      val parent = Node('\0', left.freq + right.freq, left, right)
      pq.enqueue(parent)
    }
    pq.dequeue()
  }


  val input = List(('A', 12), ('B', 23), ('C', 8), ('D', 17))

  println(huffman(input))

  val input2 = List(('a', 45), ('b', 13), ('c', 12), ('d', 16), ('e', 9), ('f', 5))

  println(huffman(input2))
```
