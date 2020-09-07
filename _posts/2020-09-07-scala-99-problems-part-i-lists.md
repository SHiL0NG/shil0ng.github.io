---
title: Scala Lists
layout: post
---

Exercise solutions for `Scala 99 Problems, Part I Lists`

## 1. Find the last element of a list.

Example:
```
scala> last(List(1, 1, 2, 3, 5, 8))
res0: Int = 8
```
Note that here I've chosen `Option[A]` as the return type instead of `null` or `throw new Exception`

- reverse and take head, pattern match


```scala
def last[A](l: List[A]): Option[A] = l match {
    case Nil => None
    case _ => Some(l.reverse.head)
}
last(List(1,2,3))
last(List())

```

- recursion, pattern match


```scala
@scala.annotation.tailrec
def last[A](l: List[A]): Option[A] = l match {
    case Nil => None
    case x::Nil => Some(x)
    case _::xs => last(xs)
}
last(List(1,2,3))
last(List())
```

- by index


```scala
def last[A](l: List[A]): Option[A] = l match {
    case Nil => None
    case _ => Some(l(l.size - 1))
}
last(List(1,2,3))
last(List())
```

## 2. Find the last but one element of a list.

Example:

```
scala> penultimate(List(1, 1, 2, 3, 5, 8))
res0: Int = 5
```
Again, here I will use `Option[A]` as the return type instead of the plain integer or whatever

- Intuitive solution


```scala
def penultimate[A](l: List[A]): Option[A] = if (l.size < 2) None else Some(l.reverse.drop(1).head)
// or we can change Some(l.reverse.drop(1).take) to Some(l(l.size - 2))
penultimate(List(1, 1, 2, 3, 5, 8))
penultimate(List())
```

## 3. Find the Kth element of a list.

By convention, the first element in the list is element 0.
Example:

```
scala> nth(2, List(1, 1, 2, 3, 5, 8))
res0: Int = 2
```

- intuitive, by index


```scala
def nth[A](n: Int, l: List[A]): Option[A] = if (n < 0 || l.size < n) None else Some(l(n))
nth(2, List(1, 1, 2, 3, 5, 8))
nth(10, List(1, 1, 2, 3, 5, 8))
nth(-2, List(1, 1, 2, 3, 5, 8))
```

- recursion, take head


```scala
@scala.annotation.tailrec
def nth[A](n: Int, l: List[A]): Option[A] = l match {
    case x::_ if (n == 0) => Some(x)
    case _::xs if (n > 0) => nth(n - 1, xs)
    case _ => None
}
nth(2, List(1, 1, 2, 3, 5, 8))
nth(0, List())
nth(10, List(1, 1, 2, 3, 5, 8))
nth(-2, List(1, 1, 2, 3, 5, 8))
```

## 4. Find the number of elements of a list.
Example:

```
scala> length(List(1, 1, 2, 3, 5, 8))
res0: Int = 6
```

- intuitive, use size


```scala
def length[T](a: List[T]): Int = a.size
length(Nil)
length(List(1,2,3))
```

- recursion


```scala
def length[A](a: List[A]): Int = a match {
    case Nil => 0
    case _::xs => 1 + length(xs)
}
length(Nil)
length(List(1,2,3))
```

- fold 


```scala
def length[A](a: List[A]): Int = a.foldLeft(0)((len, _) => len + 1)
length(Nil)
length(List(1,2,3))
```

## 5. Reverse a list.
Example:
```
scala> reverse(List(1, 1, 2, 3, 5, 8))
res0: List[Int] = List(8, 5, 3, 2, 1, 1)
```

- intuitive, build in function `reverse`


```scala
def reverse[T](a: List[T]): List[T] = a.reverse

reverse(List())
reverse(List(1,2,3,4,5))
```

- recursion, pattern matching  


```scala
def reverse[A](a: List[A]): List[A] = a match {
    case Nil => Nil
    case x::xs => reverse(xs) :+ x
}

reverse(List())
reverse(List(1,2,3,4,5))
```

- fold 


```scala
def reverse[A](a: List[A]): List[A] = a.foldLeft(List[A]())((l, e) => e::l)
reverse(List())
reverse(List(1,2,3,4,5))
```

## 6. Find out whether a list is a palindrome.
Example:

```
scala> isPalindrome(List(1, 2, 3, 2, 1))
res0: Boolean = true
```

- intuitive, list equal to reverse


```scala
def isPalindrome[A](l: List[A]): Boolean = l == l.reverse
isPalindrome(List())
isPalindrome(List(1,2,3,2,1))
isPalindrome(List('a','b','c'))
```

- two pointers


```scala
def isPalindrome[A](l: List[A]): Boolean = {
    
    @scala.annotation.tailrec
    def helper(i: Int, j: Int): Boolean = 
        if (i >= j) true
        else (l(i) == l(j)) && helper(i + 1, j - 1)

    helper(0, l.size - 1)
}

isPalindrome(List())
isPalindrome(List(1,2,3,2,1))
isPalindrome(List('a','b','c'))

```

## 7. Flatten a nested list structure.

Example:
```
scala> flatten(List(List(1, 1), 2, List(3, List(5, 8))))
res0: List[Any] = List(1, 1, 2, 3, 5, 8)
```

- use flatMap


```scala
def flatten(l: List[Any]): List[Any] = l.flatMap {
    case x: List[_] => flatten(x)
    case x: Any => List(x)
}
flatten(List(1, List(2,3, List(4,5,6), List(7)), List(8,9)))
```

- solve with new types
A `List[Any]` does not make me feel very comfortable. The parameter is obviously not of type `List[A]`. We give it a new type. Let's say `Element[T]` which could be a single element of `T` or a collection of `Element[T]`. This does not solve exactly the problem, but it has its own advantage: type safe.


```scala
def flatten[T](elem: Element[T]): List[T] = elem match {
    case x: SE[T] => List(x.value)
    case xs: CE[T] => xs.elements.flatMap(ne => flatten(ne)).toList
}


sealed trait Element[T]
case class SE[T](value: T) extends Element[T] // single
case class CE[T](elements: Element[T]*) extends Element[T] // composite

implicit def toSE[T](v: T): SE[T] = SE(v)

flatten(CE(2, CE(3,4, CE(4,5,6), 7, CE(8))))

```

## 8. Eliminate consecutive duplicates of list elements.
If a list contains repeated elements they should be replaced with a single copy of the element. The order of the elements should not be changed.
Example:
```sh
scala> compress(List('a, 'a, 'a, 'a, 'b, 'c, 'c, 'a, 'a, 'd, 'e, 'e, 'e, 'e))
res0: List[Symbol] = List('a, 'b, 'c, 'a, 'd, 'e)
```

- fold


```scala
// if the currently visiting element is the same as the list tail element, ignore it, otherwise append to list
def compress(list: List[Symbol]): List[Symbol] = list match {
  case Nil => List()
  case _ => list.tail.foldLeft(List[Symbol](list.head))((l, e) => {if (e equals l.last) l else l :+ e })
}
compress(List('a, 'a, 'a, 'a, 'b, 'c, 'c, 'a, 'a, 'd, 'e, 'e, 'e, 'e))
compress(List[Symbol]())
compress(List('a))
```

- recursion
Use the partial result, the last visited symbol and the remaining list as parameters of the recursion function


```scala
def compress[T](list: List[T]): List[T] = list match {
  case Nil => List()
  case x::xs => compress(List(x), x, xs)
}


@scala.annotation.tailrec
def compress[T](result: List[T], last: T, remain: List[T]): List[T] = remain match {
  case Nil => result
  case x::xs =>
    if (x == last) compress(result, x, xs)
    else compress(result :+ x, x, xs)
}

compress(List('a, 'a, 'a, 'a, 'b, 'c, 'c, 'a, 'a, 'd, 'e, 'e, 'e, 'e))
compress(List[Symbol]())
compress(List('a))
```

## 9. Pack consecutive duplicates of list elements into sublists.
If a list contains repeated elements they should be placed in separate sublists.
Example:

```
scala> pack(List('a, 'a, 'a, 'a, 'b, 'c, 'c, 'a, 'a, 'd, 'e, 'e, 'e, 'e))
res0: List[List[Symbol]] = List(List('a, 'a, 'a, 'a), List('b), List('c, 'c), List('a, 'a), List('d), List('e, 'e, 'e, 'e))
```

- recursion
Similar to the previous problem, we put the partial result, the pack under way and also the remaining list as the parameters of the recursion function. 


```scala
def pack(list: List[Symbol]): List[List[Symbol]] = pack(List[List[Symbol]](), List[Symbol](), list)

def pack(lists: List[List[Symbol]], list: List[Symbol], rem: List[Symbol]): List[List[Symbol]] = rem match {
    case Nil => lists :+ list
    case x::xs if (list.isEmpty || list.contains(x)) => pack(lists, list :+ x, xs)
    case _ => pack(lists :+ list, List(), rem)
}
pack(List('a, 'a, 'a, 'a, 'b, 'c, 'c, 'a, 'a, 'd, 'e, 'e, 'e, 'e))
```

## 10. Run-length encoding of a list.
Use the result of problem P09 to implement the so-called run-length encoding data compression method. Consecutive duplicates of elements are encoded as tuples (N, E) where N is the number of duplicates of the element E.
Example:
```
scala> encode(List('a, 'a, 'a, 'a, 'b, 'c, 'c, 'a, 'a, 'd, 'e, 'e, 'e, 'e))
res0: List[(Int, Symbol)] = List((4,'a), (1,'b), (2,'c), (2,'a), (1,'d), (4,'e))
```
- reuse `pack` method + `map`


```scala
def pack[T](lists: List[List[T]], list: List[T], rem: List[T]): List[List[T]] = rem match {
    case Nil => lists :+ list
    case x::xs if (list.isEmpty || list.contains(x)) => pack(lists, list :+ x, xs)
    case _ => pack(lists :+ list, List(), rem)
}

def encode[T](symbols: List[T]): List[(Int, T)] = pack(List[List[T]](), List[T](), symbols).map(l => (l.size, l.head))

encode(List('a, 'a, 'a, 'a, 'b, 'c, 'c, 'a, 'a, 'd, 'e, 'e, 'e, 'e))
```

## 11. Modified run-length encoding.
Modify the result of problem P10 in such a way that if an element has no duplicates it is simply copied into the result list. Only elements with duplicates are transferred as (N, E) terms.
Example:
```
scala> encodeModified(List('a, 'a, 'a, 'a, 'b, 'c, 'c, 'a, 'a, 'd, 'e, 'e, 'e, 'e))
res0: List[Any] = List((4,'a), 'b, (2,'c), (2,'a), 'd, (4,'e))
```

- reuse `pack`
Here we changed the implementation of `pack` using the built-in `span` method, then we apply `map` to the `pack` result 


```scala
def pack[T](lists: List[List[T]], list: List[T]): List[List[T]] = list match {
    case Nil => lists
    case _ => {
        val (l, r) = list span (_ == list.head)
        pack(lists :+ l, r)
    }
} 

pack(List[List[Symbol]](), List('a, 'a, 'a, 'a, 'b, 'c, 'c, 'a, 'a, 'd, 'e, 'e, 'e, 'e))

def encodeModified[T](list: List[T]): List[Any] = 
    pack(List[List[T]](), list).map(l => if (l.size > 1) (l.size, l.head) else l.head)

encodeModified(List('a, 'a, 'a, 'a, 'b, 'c, 'c, 'a, 'a, 'd, 'e, 'e, 'e, 'e))
```

- be type safe

As always, I do not like `Any`, we could use `List[Either[(Int, T), T]]` as the return type.


```scala
def encodeModified[T](list: List[T]): List[Either[(Int, T), T]] = 
    pack(List[List[T]](), list).map(l => l match {
        case e if (e.size > 1) => Left((e.size, e.head))
        case e => Right(e.head)
    })

encodeModified(List('a, 'a, 'a, 'a, 'b, 'c, 'c, 'a, 'a, 'd, 'e, 'e, 'e, 'e))
```

## 12. Decode a run-length encoded list.
Given a run-length code list generated as specified in problem P10, construct its uncompressed version.
Example:
```
scala> decode(List((4, 'a), (1, 'b), (2, 'c), (2, 'a), (1, 'd), (4, 'e)))
res0: List[Symbol] = List('a, 'a, 'a, 'a, 'b, 'c, 'c, 'a, 'a, 'd, 'e, 'e, 'e, 'e)
```

- use flatMap to map each tuple to a list of symbols


```scala
def decode[T](l: List[(Int, T)]): List[T] = l flatMap {
    e => List.fill(e._1)(e._2)
}

decode(List((4, 'a), (1, 'b), (2, 'c), (2, 'a), (1, 'd), (4, 'e)))
```

## 13. Run-length encoding of a list (direct solution).
Implement the so-called run-length encoding data compression method directly. I.e. don't use other methods you've written (like P09's pack); do all the work directly.
Example:
```
scala> encodeDirect(List('a, 'a, 'a, 'a, 'b, 'c, 'c, 'a, 'a, 'd, 'e, 'e, 'e, 'e))
res0: List[(Int, Symbol)] = List((4,'a), (1,'b), (2,'c), (2,'a), (1,'d), (4,'e))
```



```scala
def encodeDirect[T](l: List[T]): List[(Int, T)] = l match {
    case Nil => List()
    case _ => {
        val (a, b) = l span (_ == l.head)
        (a.size, a.head)::encodeDirect(b)
    }
}
encodeDirect(List('a, 'a, 'a, 'a, 'b, 'c, 'c, 'a, 'a, 'd, 'e, 'e, 'e, 'e))
```

## 14. Duplicate the elements of a list.
Example:
```
scala> duplicate(List('a, 'b, 'c, 'c, 'd))
res0: List[Symbol] = List('a, 'a, 'b, 'b, 'c, 'c, 'c, 'c, 'd, 'd)
```


```scala
def duplicate[T](list: List[T]): List[T] = {
    list.flatMap(a => List(a, a))
}

duplicate(List('a, 'b, 'c, 'c, 'd))
```

## 15. Duplicate the elements of a list a given number of times.
Example:
```
scala> duplicateN(3, List('a, 'b, 'c, 'c, 'd))
res0: List[Symbol] = List('a, 'a, 'a, 'b, 'b, 'b, 'c, 'c, 'c, 'c, 'c, 'c, 'd, 'd, 'd)
```

- flatMap


```scala
def duplicateN[T](n: Int, list: List[T]): List[T] = {
    list.flatMap(e => List.fill(n)(e))
}
duplicateN(3, List('a, 'b, 'c, 'c, 'd))
```

## 16. Drop every Nth element from a list.
Example:
```
scala> drop(3, List('a, 'b, 'c, 'd, 'e, 'f, 'g, 'h, 'i, 'j, 'k))
res0: List[Symbol] = List('a, 'b, 'd, 'e, 'g, 'h, 'j, 'k)
```


```scala
def drop[T](n: Int, list: List[T]): List[T] = (for (i <- 1 to list.size if i % n != 0) yield list(i - 1)).toList
drop(3, List('a, 'b, 'c, 'd, 'e, 'f, 'g, 'h, 'i, 'j, 'k))
drop(1, List())
drop(5, List('a, 'b, 'c, 'd, 'e, 'f, 'g, 'h, 'i, 'j, 'k))
```

## 17.  Split a list into two parts.
The length of the first part is given. Use a Tuple for your result.
Example:
```
scala> split(3, List('a, 'b, 'c, 'd, 'e, 'f, 'g, 'h, 'i, 'j, 'k))
res0: (List[Symbol], List[Symbol]) = (List('a, 'b, 'c),List('d, 'e, 'f, 'g, 'h, 'i, 'j, 'k))
```
- use `splitAt`


```scala
def split[T](n: Int, list: List[T]): (List[T], List[T]) = list.splitAt(n)
split(3, List('a, 'b, 'c, 'd, 'e, 'f, 'g, 'h, 'i, 'j, 'k))
```

- recursion


```scala
def split[T](n: Int, list: List[T]): (List[T], List[T]) = {
    def split(first: List[T], n: Int, second: List[T]): (List[T], List[T]) = {
        if (n == 0) (first, second)
        else split(first :+ second.head, n - 1, second.tail)
    }
    split(List(), n, list)
}

split(3, List('a, 'b, 'c, 'd, 'e, 'f, 'g, 'h, 'i, 'j, 'k))
```

## 18. Extract a slice from a list.
Given two indices, I and K, the slice is the list containing the elements from and including the Ith element up to but not including the Kth element of the original list. Start counting the elements with 0.
Example:
```
scala> slice(3, 7, List('a, 'b, 'c, 'd, 'e, 'f, 'g, 'h, 'i, 'j, 'k))
res0: List[Symbol] = List('d, 'e, 'f, 'g)
```

- built-in slice


```scala
def slice[T](from: Int, to: Int, list: List[T]): List[T] = list.slice(from, to)
slice(3, 7, List('a, 'b, 'c, 'd, 'e, 'f, 'g, 'h, 'i, 'j, 'k))
```

- drop & take


```scala
def slice[T](from: Int, to: Int, list: List[T]): List[T] = list.drop(from).take(to - from)
slice(3, 7, List('a, 'b, 'c, 'd, 'e, 'f, 'g, 'h, 'i, 'j, 'k))
```

### 19. Rotate a list N places to the left.
Examples:
```
scala> rotate(3, List('a, 'b, 'c, 'd, 'e, 'f, 'g, 'h, 'i, 'j, 'k))
res0: List[Symbol] = List('d, 'e, 'f, 'g, 'h, 'i, 'j, 'k, 'a, 'b, 'c)

scala> rotate(-2, List('a, 'b, 'c, 'd, 'e, 'f, 'g, 'h, 'i, 'j, 'k))
res1: List[Symbol] = List('j, 'k, 'a, 'b, 'c, 'd, 'e, 'f, 'g, 'h, 'i)
```
- recursion, each time rotate by 1 element


```scala
def rotate[T](n: Int, list: List[T]): List[T] = {
  if (n >= 0) rotateLeft(n, list)
  else rotateLeft(list.size + n, list)
}
// if (n >= 0) list.drop(n) ::: list.take(n)
@scala.annotation.tailrec
def rotateLeft[T](n: Int, list: List[T]): List[T] =  {
  if (n == 0) list
  else rotateLeft(n - 1, list.tail :+ list.head)
}

rotate(3, List('a, 'b, 'c, 'd, 'e, 'f, 'g, 'h, 'i, 'j, 'k))
rotate(-2, List('a, 'b, 'c, 'd, 'e, 'f, 'g, 'h, 'i, 'j, 'k))
```

- split


```scala
def rotate[T](n: Int, list: List[T]): List[T] = {
    val m = if (n < 0) list.size + n else n
    val (l, r) = list.splitAt(m)
    r:::l
}
rotate(3, List('a, 'b, 'c, 'd, 'e, 'f, 'g, 'h, 'i, 'j, 'k))
rotate(-2, List('a, 'b, 'c, 'd, 'e, 'f, 'g, 'h, 'i, 'j, 'k))
```

## 20. Remove the Kth element from a list.
Return the list and the removed element in a Tuple. Elements are numbered from 0.
Example:
```
scala> removeAt(1, List('a, 'b, 'c, 'd))
res0: (List[Symbol], Symbol) = (List('a, 'c, 'd),'b)
```

- take, drop


```scala
def removeAt[T](k: Int, list: List[T]): (List[T], T) = 
    // assert k < list.size
    (list.take(k) ++ list.drop(k + 1), list(k))

removeAt(1, List('a, 'b, 'c, 'd))
```

- recursion


```scala
def removeAt[T](k: Int, list: List[T]): (List[T], T) = {
    def helper(left: List[T], n: Int, right: List[T]): (List[T], T) = {
        // if right == Nil throw exception
        if (n == 0) (left ++ right.tail, right.head)
        else helper(left :+ right.head, n - 1, right.tail)
    }
    helper(List(), k, list)
}

removeAt(1, List('a, 'b, 'c, 'd))
removeAt(0, List('a, 'b, 'c, 'd))
```

## 21. Insert an element at a given position into a list.
Example:
```
scala> insertAt('new, 1, List('a, 'b, 'c, 'd))
res0: List[Symbol] = List('a, 'new, 'b, 'c, 'd)
```


```scala
def insertAt[T](elem: T, n: Int, list: List[T]): List[T] = list.take(n) ++ List(elem) ++ list.drop(n)
insertAt('new, 1, List('a, 'b, 'c, 'd))
```

## 22. Create a list containing all integers within a given range.
Example:
```
scala> range(4, 9)
res0: List[Int] = List(4, 5, 6, 7, 8, 9)
```

- yield


```scala
def range(start: Int, end:Int): List[Int] = {
    (for (i <- start to end) yield i).toList
}
range(4, 9)
```

- recursion


```scala
def range(start: Int, end:Int): List[Int] = {
    if (start == end) List()
    else start:: range(start + 1, end)
}
range(4, 9)
```


```scala
def range(start: Int, end: Int): List[Int] = {
    @scala.annotation.tailrec
    def rangeRecursive(result: List[Int], start: Int, end: Int): List[Int] = {
        if (start == end) result:+ end
        else rangeRecursive(result :+ start, start + 1, end)
    }
    rangeRecursive(List(), start, end);
}
range(4, 9)
```

## 23. Extract a given number of randomly selected elements from a list.
Example:
```
scala> randomSelect(3, List('a, 'b, 'c, 'd, 'f, 'g, 'h))
res0: List[Symbol] = List('e, 'd, 'a)
```
Hint: Use the solution to problem P20



```scala
import scala.util.Random
def removeAt[T](k: Int, list: List[T]): (List[T], T) = 
    // assert k < list.size
    (list.take(k) ++ list.drop(k + 1), list(k))

def randomSelect[T](n: Int, list: List[T]):List[T] =  
    if (n <= 0 || list.size == 0) List()
    else {
        val (l, x) = removeAt(Random.nextInt(list.size), list)
        x::randomSelect(n - 1, l) // same element cannot be re-selected
    }

randomSelect(3, List('a, 'b, 'c, 'd, 'f, 'g, 'h))
randomSelect(7, List('a, 'b, 'c, 'd, 'f, 'g, 'h))
randomSelect(8, List('a, 'b, 'c, 'd, 'f, 'g, 'h))
```

## 24. Lotto: Draw N different random numbers from the set 1..M.
Example:
```
scala> lotto(6, 49)
res0: List[Int] = List(23, 1, 17, 33, 21, 37)
```
- reuse `randomSelect` from Problem 23


```scala
def lotto(N: Int, M: Int): List[Int] = randomSelect(N, (1 to M).toList)
lotto(6, 49)
```

## 25. Generate a random permutation of the elements of a list.
Hint: Use the solution of problem P23.
Example:
```
scala> randomPermute(List('a, 'b, 'c, 'd, 'e, 'f))
res0: List[Symbol] = List('b, 'a, 'd, 'c, 'e, 'f)
```


```scala
def randomPermute[T](list: List[T]): List[T] = randomSelect(list.size, list)
randomPermute(List('a, 'b, 'c, 'd, 'e, 'f))
```

## 26. Generate the combinations of K distinct objects chosen from the N elements of a list.
In how many ways can a committee of 3 be chosen from a group of 12 people? We all know that there are C(12,3) = 220 possibilities (C(N,K) denotes the well-known binomial coefficient). For pure mathematicians, this result may be great. But we want to really generate all the possibilities.
Example:
```
scala> combinations(3, List('a, 'b, 'c, 'd, 'e, 'f))
res0: List[List[Symbol]] = List(List('a, 'b, 'c), List('a, 'b, 'd), List('a, 'b, 'e), ...
```

- use built-in `combinations(Int)`


```scala
def combinations[T](n: Int, list: List[T]): List[List[T]] = list.combinations(n).toList
combinations(3, List('a, 'b, 'c, 'd, 'e, 'f))
```

- recursion


```scala
def combinations[T](n: Int, list: List[T]): List[List[T]] = {
    import scala.collection.mutable.ListBuffer
    
    var lists = ListBuffer[List[T]]()
    // pick: min index feasible to choose from
    def helper(combo: List[T], k: Int, pick: Int): Unit = {
        if (k == 0) lists = lists :+ combo
        else {
            for (i <- pick until list.size)
                helper(combo :+ list(i), k - 1, i + 1)
        }
    }
    
    helper(List(), n, 0)
    
    lists.toList
}
combinations(3, List('a, 'b, 'c, 'd, 'e, 'f))
combinations(6, List('a, 'b, 'c, 'd, 'e, 'f))
combinations(10, List('a, 'b, 'c, 'd, 'e, 'f))
```

- recursion, depending whether or not we use the current element 


```scala
def combinations[T](n: Int, list: List[T]): List[List[T]] = list match {
  case _ :: _ if n == 1 => list.map(List(_)) // end of recursion
  case x::xs => combinations(n - 1, xs).map(x::_) ::: combinations(n, xs) // choose x ::: not choose x
  case Nil => Nil
}

combinations(3, List('a, 'b, 'c, 'd, 'e, 'f))
combinations(6, List('a, 'b, 'c, 'd, 'e, 'f))
combinations(10, List('a, 'b, 'c, 'd, 'e, 'f))
```

- another possible recursive implementation


```scala
  def combinations[T](n: Int, list: List[T]): List[List[T]] =
    combinations(List(), n, list)

  def combinations[T](partial: List[T], n: Int, list: List[T]): List[List[T]] = {
    if (n > list.size) List()
    else if (n == 0) List(partial)
    else  {
      combinations(partial :+ list.head, n - 1, list.tail) ++
      combinations(partial, n, list.tail)
    }
  }
```

## 27.  Group the elements of a set into disjoint subsets.
a) In how many ways can a group of 9 people work in 3 disjoint subgroups of 2, 3 and 4 persons? Write a function that generates all the possibilities.
Example:
```
scala> group3(List("Aldo", "Beat", "Carla", "David", "Evi", "Flip", "Gary", "Hugo", "Ida"))
res0: List[List[List[String]]] = List(List(List(Aldo, Beat), List(Carla, David, Evi), List(Flip, Gary, Hugo, Ida)), ...
```
b) Generalize the above predicate in a way that we can specify a list of group sizes and the predicate will return a list of groups.

Example:
```
scala> group(List(2, 2, 5), List("Aldo", "Beat", "Carla", "David", "Evi", "Flip", "Gary", "Hugo", "Ida"))
res0: List[List[List[String]]] = List(List(List(Aldo, Beat), List(Carla, David), List(Evi, Flip, Gary, Hugo, Ida)), ...
```
Note that we do not want permutations of the group members; i.e. ((Aldo, Beat), ...) is the same solution as ((Beat, Aldo), ...). However, we make a difference between ((Aldo, Beat), (Carla, David), ...) and ((Carla, David), (Aldo, Beat), ...).

You may find more about this combinatorial problem in a good book on discrete mathematics under the term "multinomial coefficients".

- make combinations for each element of the `groups`



```scala
  def combinations[T](n: Int, list: List[T]): List[List[T]] = list.combinations(n).toList

  def group[T](groups: List[Int], elements: List[T]): List[List[List[T]]] = groups match {
    // assert groups.sum == elements.size
    case Nil => Nil
    case _ => {
      val combos = combinations(groups.head, elements)
      combos.foldLeft(List[List[List[T]]]()) {
        (groupings, combo) => {
            val subgroupings = group(groups.tail, elements.diff(combo)) // recursion, all possible subgroupings without group.head and combo
            subgroupings match {
                case Nil => groupings ++ List(List(combo))
                case _ => groupings ++ subgroupings.map(combo::_) // add combo to the subgrouping, add subgroupings containing combo to final result
            }    
        }
      }
    }
  }

group(List(2, 2, 5), List("Aldo", "Beat", "Carla", "David", "Evi", "Flip", "Gary", "Hugo", "Ida")).size

```

The final result size should be `9!/(2!2!5!) = 756`

## 28. Sorting a list of lists according to length of sublists.
a) We suppose that a list contains elements that are lists themselves. The objective is to sort the elements of the list according to their length. E.g. short lists first, longer lists later, or vice versa.
Example:
```
scala> lsort(List(List('a, 'b, 'c), List('d, 'e), List('f, 'g, 'h), List('d, 'e), List('i, 'j, 'k, 'l), List('m, 'n), List('o)))
res0: List[List[Symbol]] = List(List('o), List('d, 'e), List('d, 'e), List('m, 'n), List('a, 'b, 'c), List('f, 'g, 'h), List('i, 'j, 'k, 'l))
```
b) Again, we suppose that a list contains elements that are lists themselves. But this time the objective is to sort the elements according to their length frequency; i.e. in the default, sorting is done ascendingly, lists with rare lengths are placed, others with a more frequent length come later.

Example:
```
scala> lsortFreq(List(List('a, 'b, 'c), List('d, 'e), List('f, 'g, 'h), List('d, 'e), List('i, 'j, 'k, 'l), List('m, 'n), List('o)))
res1: List[List[Symbol]] = List(List('i, 'j, 'k, 'l), List('o), List('a, 'b, 'c), List('f, 'g, 'h), List('d, 'e), List('d, 'e), List('m, 'n))
```
Note that in the above example, the first two lists in the result have length 4 and 1 and both lengths appear just once. The third and fourth lists have length 3 and there are two lists of this length. Finally, the last three lists have length 2. This is the most frequent length.


```scala
def lsort[T](list: List[List[T]]): List[List[T]] = list.sortWith((l1, l2) => l1.size < l2.size)
def lsortFreq[T](list: List[List[T]]): List[List[T]] = {
  val frequencies = scala.collection.mutable.Map[Int, Int]().withDefaultValue(0)
  list.foreach(l => frequencies.update(l.size, frequencies(l.size) + 1))
  list.sortWith((l1, l2) => frequencies(l1.size) < frequencies(l2.size))
}
```

