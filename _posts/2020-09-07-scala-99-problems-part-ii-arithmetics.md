---
title: Scala Arithmetics
layout: post
---

Exercise solutions for `Scala 99 Problems, Part II Arithmetics.`

# 31. Determine whether a given integer number is prime.
```
scala> 7.isPrime
res0: Boolean = true
```


```scala
object P31 {
  import java.lang.Math

  implicit class PrimeChecker(n: Int) {
    def isPrime(): Boolean = {
      if (n < 2) false
      else !(2 to Math.sqrt(n).toInt).exists(n % _ == 0)
    }
  }
    
}

import P31._

7.isPrime
13.isPrime
81.isPrime
```

## 32. Determine the greatest common divisor of two positive integer numbers.
Use Euclid's algorithm.
```
scala> gcd(36, 63)
res0: Int = 9
```


```scala
  def gcd(a: Int, b: Int): Int = {
    if (b == 0) a
    else gcd(b,  a % b)
  }
  gcd(93, 42)

  gcd(0, 10)
```

## 33. Determine whether two positive integer numbers are coprime.
Two numbers are coprime if their greatest common divisor equals 1.

```
scala> 35.isCoprimeTo(64)
res0: Boolean = true
```


```scala

  def gcd(a: Int, b: Int): Int = {
    if (b == 0) a
    else gcd(b,  a % b)
  }
  
  implicit def fromIntToCoPrimeValidator(v: Int) = CoPrimeValidator(v)
  
  case class CoPrimeValidator(v: Int) {
    def isCoPrimeTo(another: Int): Boolean = 1 == gcd(v, another)
  }


  12.isCoPrimeTo(45)
  12.isCoPrimeTo(35)
```

## 34. Calculate Euler's totient function phi(m).
Euler's so-called totient function phi(m) is defined as the number of positive integers r (1 <= r <= m) that are coprime to m.
```
scala> 10.totient
res0: Int = 4
```


```scala
  implicit def integerToEulerTotientCal(n: Int): EulerTotientCal = EulerTotientCal(n)

  case class EulerTotientCal(n: Int) {
    def totient: Int = (1 to n).count(_.isCoPrimeTo(n))
  }

  10.totient
  315.totient
```

## 35. Determine the prime factors of a given positive integer.
Construct a flat list containing the prime factors in ascending order.

```
scala> 315.primeFactors
res0: List[Int] = List(3, 3, 5, 7)
```



```scala
  implicit def integerToPrimeFactor(n: Int): PrimeFactor = PrimeFactor(n)

  case class PrimeFactor(n: Int) {
    def primeFactors: List[Int] = primeFactors(2, n)

    def primeFactors(f: Int, n: Int): List[Int] = {
      if (n == 1) List()
      else if (n % f == 0) f :: primeFactors(f, n/f)
      else primeFactors(f + 1, n)
    }
  }
  315.primeFactors
```

## 36. Determine the prime factors of a given positive integer (2).
Construct a list containing the prime factors and their multiplicity.

```
scala> 315.primeFactorMultiplicity
res0: List[(Int, Int)] = List((3,2), (5,1), (7,1))
```
Alternately, use a Map for the result.
```
scala> 315.primeFactorMultiplicity
res0: Map[Int,Int] = Map(3 -> 2, 5 -> 1, 7 -> 1)
```



```scala
  implicit def integerToPrimeFactorsMap(n: Int): PrimeFactorsMap = PrimeFactorsMap(n)

  case class PrimeFactorsMap(n: Int) {
    def primeFactorsMultiplicity: Map[Int, Int] =
      // reuse primeFactors
      n.primeFactors.groupBy(identity).mapValues(_.size)

    def primeFactorsMultiplicity2: Map[Int, Int] =
      n.primeFactors.foldLeft(Map[Int, Int]() withDefaultValue 0)((m, i) => m.updated(i, m(i) + 1))
  }

  315.primeFactorsMultiplicity
  315.primeFactorsMultiplicity2
```

### 37. Calculate Euler's totient function phi(m) (improved).



```scala
  implicit def intToEulerTotientImproved(v: Int): EulersTotientImproved = EulersTotientImproved(v)

  case class EulersTotientImproved(value: Int) {
    def totientImproved: Int = {
      import java.lang.Math
      value.primeFactorsMultiplicity.foldLeft(1)((r, t) => r * (t._1 - 1) * Math.pow(t._1, t._2 - 1).toInt)
    }
  }

  315.totientImproved
```

## 39.  A list of prime numbers.
Given a range of integers by its lower and upper limit, construct a list of all prime numbers in that range.
```
scala> listPrimesinRange(7 to 31)
res0: List[Int] = List(7, 11, 13, 17, 19, 23, 29, 31)
```


```scala
import P31._ 

def listPrimesInRange(range: Range): List[Int] = range.filter(_.isPrime).toList

listPrimesInRange(1 to 100)
```

## 40. Goldbach's conjecture.
Goldbach's conjecture says that every positive even number greater than 2 is the sum of two prime numbers. E.g. 28 = 5 + 23. It is one of the most famous facts in number theory that has not been proved to be correct in the general case. It has been numerically confirmed up to very large numbers (much larger than Scala's Int can represent). Write a function to find the two prime numbers that sum up to a given even integer.
```
scala> 28.goldbach
res0: (Int, Int) = (5,23)
```


```scala
object P40 {
 implicit class GoldbachInt(value: Int) {
    import P31._
    def goldbach: (Int, Int) = {
      (2 to value).find(p => p.isPrime && (value -p).isPrime).map(p => (p, value - p)).get
    }
  }
}

import P40._

28.goldbach

```

### 41. A list of Goldbach compositions.
Given a range of integers by its lower and upper limit, print a list of all even numbers and their Goldbach composition.
```
scala> printGoldbachList(9 to 20)
10 = 3 + 7
12 = 5 + 7
14 = 3 + 11
16 = 3 + 13
18 = 5 + 13
20 = 3 + 17
```
In most cases, if an even number is written as the sum of two prime numbers, one of them is very small. Very rarely, the primes are both bigger than, say, 50. Try to find out how many such cases there are in the range 2..3000.

Example (minimum value of 50 for the primes):
```
scala> printGoldbachListLimited(1 to 2000, 50)
992 = 73 + 919
1382 = 61 + 1321
1856 = 67 + 1789
1928 = 61 + 1867
```


```scala
  def printGoldbachList(range: Range): List[(Int, Int)] = {
    import P40._
    range.filter(p => p > 2 && p % 2 == 0).map(v => v.goldbach).toList
  }

  def printGoldbachListLimited(range: Range, limit: Int): List[(Int, Int)] = {
    printGoldbachList(range).filter(a => a._1 > limit)
  }
  printGoldbachListLimited(1 to 2000, 50)
  printGoldbachList(9 to 20)
```


```scala

```
