---
title: Programming Notes
description: A manual contains reference information about practical programming problems I met since just first getting into coding.
tags: Programming
---

Introduction
============

<p>This manual contains reference information about practical programming problems I met since just first getting
 into coding. The manual is divided into the
following sections:</p>
<ul>
<li><a href="http://www.ee.ic.ac.uk/hp/staff/dmb/matrix/index.html">Main Index</a>: Alphabetical index of all
entries.</li>
<li><a href="http://www.ee.ic.ac.uk/hp/staff/dmb/matrix/property.html">Java Traps</a> : Problems I met while programming in Java.</li>
<li><a href="http://www.ee.ic.ac.uk/hp/staff/dmb/matrix/property.html">Matlab Quick Reference</a> : Functions and snippets I find useful.</li>
<li><a href="http://www.ee.ic.ac.uk/hp/staff/dmb/matrix/property.html">Haskell Quick Reference</a> : Functions and snippets I find useful.</li>
<li><a href="http://www.ee.ic.ac.uk/hp/staff/dmb/matrix/gfl.html">GNU Free Documentation License</a></li>
</ul>

C++ Quick Reference
===================

## Pointer Syntax ##
    int *p; // Defines pointer.
    p = &q; // Sets p to address of q.
    v = *p; // Set v to value of q.
    Foo *f = new Foo(); // Initializes f.
    int k = f->x; // Sets k equal to the value of f's member variable.
## C++ Class Syntax ##
    class MyClass {
        private:
            double var;
        public:
            MyClass(double v) {var = v; }
    };
    double Complex::Update(double v) {
        var = v; return v;
    }
    
## C++ vs Java ##
1. Java runs in a virtual machine.
2. C++ natively supports unsigned arithmetic.
3. In java, parameters are always passed by value (or, with objects, their references are passed by value). In C++, parameters can be passed by value, pointer, or by reference.
4. Java has built-in garbage collection.
5. C++ allows operator overloading.
6. C++ allows multiple inheritance of classes.

Java Quick Reference
====================

## Classes & Interfaces ##
    public static foid main(String args[]) {...}
    interface Foo {
        void abc();
    }
    class Foo extends Bar implements Foo {...}
    
## final: ##
1. Class: Can not be sub-classed.
2. Method: Can not be overridden.
3. Variable: Can not be changed.

## static: ##
1. Method: Class method. Called with Foo.Dolt()
2. Variable: Class variable. Has only one copy and is accessed through the calss name.

## abstract: ##
1. Class: Contains abstract methods. Can not be instantiated.
2. Interface: All interfaces are implicitly abstract. This modifier is optional.
3. Method: Method without a body. Class must also be abstract.

Java Snippets
=============

## Arrays and Strings ##

### Hash Tables ###
	public HashMap<Integer, Student> buildMap(Student[] students) {
		HashMap<Interger, Student> map = new HashMap<Integer, Student>();
		for (Student s : students) map.put(s.getId(), s);
		return map;
	}

### StringBuffer/StringBuilder ###
	public String makeSentence(String[] words) {
		StringBuffer sentence = new StringBuffer();
		for (String w : words) sentence.append(w);
		return sentence.toString();
	}
	
### isSubString ###
An easy way of implementing isSubstring which checks if one word is a substring of another.
	
    public static boolean isSubstring(String big, String small) {
		if (big.indexOf(small) >= 0) {
			return true;
		} else {
			return false;
		}
	}
	
## Linked Lists ##
### Creating a Linked List ###

    class Node {
        Node next = null;
        int data;
        public Node(int d) {data = d; }
        void appendToTail(int d) {
            Node end = new Node(d);
            Node n = this;
            while (n.next != null) { n = n.next; }
            n.next = end;
        }
    }

### Deleting a Node from a Singly Linked List: ###

    Node deleteNode(Node head, int d) {
        Node n = head;
        if (n.data == d) {
            return head.next; /* moved head */
        }
        while (n.next != null) {
            if (n.next.data == d) {
                n.next = n.next.next;
                return head; /* head didn't change */
            }
            n = n.next;
        }
    }
    
### Get the length of a Linked List ###
    
    private static int length(LinkedListNode l) {
		if (l == null) {
			return 0;
		} else {
			return 1 + length(l.next);
		}
	}
	
## Stacks and Queues ##
### Implementing a Stack ###
    class Stack {
        Node top;
        Node pop() {
            if (top != null) {
                Object item = top.data;
                top = top.next;
                return item;
            }
            return null;
        }
        void push(Object item) {
            Node t = new Node(item);
            t.next = top;
            top = t;
        }
    }

### Implementing a Queue ###
    class Queue {
        Node first, last;
        void enqueue(Object item) {
            if (!first) {
                back = new Node(item);
                first = back;
            } else {
                back.next = new Node(item);
                back = back.next;
            }
        }
        Node dequeue(Node n) {
            if (front != null) {
                Object item = front.data;
                front = front.next;
                return item;
            }
            return null;
        }
    }
Java Traps
==========

<h3>Type Erasure</h3>
<p>I once tried to put these two methods in the same class:
<pre><code>class Test{
   void add(Set&lt;Integer&gt; ii){}
   void add(Set&lt;String&gt; ss){}
}
</code></pre>
</p>

Clojure Snippets
=============

### Counting Neighbours ###

	(defn neighbours
		[[x y]]
		(for [dx [-1 0 1] dy [-1 0 1] :when (not= 0 dx dy)]
			[(+ dx x) (+ dy y]))
	
	(defn count-neighbours
		[board loc]
		(count (filter #(get-in board %) (neighbour loc))))

Matlab Quick Reference
======================

<h3>Functions</h3>
<h4>Full: convert sparse matrix to full matrix</h4>
<p>
<code>A = full(S)</code> converts a sparse matrix <code>S</code> to full storage organization. If <code>S</code> is a full matrix, 
it is left unchanged. If <code>A</code> is full, <code>issparse(A)</code> is <code>0</code>.
</p>
<p>
<code>B = sparse(10000,10000,pi)</code> sets up a 10000-by-10000 matrix with only one nonzero element. Trying <code>full(B)</code> requires 800 megabytes of storage.
</p>
<h4>Find nonzero indices</h4>
<p><code>ind = find(X)</code> locates all nonzero elements of array <code>X</code>, and returns the linear indices of those elements in vector <code>ind</code>.</p>
<h4>Size: array dimensions</h4>
<p><code>m = size(X,dim)</code> returns the size of the dimension of <code>X</code> specified by scalar <code>dim</code>.</p>
<p> For a Java array, <code>size</code> returns the length of the Java array as the number of rows. The number of columns is always 1. 
For a Java array of arrays, the result describes only the top level array.</p>
<h4>Sum of array elements</h4>
<p><code>B = sum(A,dim)</code> sums along the dimension of <code>A</code> specified by scalar <code>dim</code>. The <code>dim</code> input is an integer value from 
<code>1</code> to <code>N</code>, where <code>N</code> is the number of dimensions in <code>A</code>. Set <code>dim</code> to <code>1</code> to compute the sum of each 
column, <code>2</code> to sum rows, etc.</p>

<h3>Snippets</h3>
<h4>Select papers written by certain authors with a map</h4>
<p><code>apapers</code> is a cell array of papers written by each author. The goal is to select the author-paper map of which the author indices are in <code>alist</code>. Using <code>sum</code> and <code>find</code>, we can easily locate the nonzero elements and select these colums with a list 
assignment.
<pre><code>
numa = length(alist);            % number of authors
totp = length(ptitles);          % total number of papers

apmap = zeros(numa,totp);
for aa = 1:numa
  apmap(aa,apapers{alist(aa)}) = 1;
end
plist = find(sum(apmap,1));      % indices of papers
apmap = apmap(:,plist);          % entry (i,j)=1 if author i wrote paper j
</code></pre>
</p>

<h4>Pick informative words</h4>
<p>Estimate informativeness by first do tf-idf, then compute entropy. <code>wpcounts</code> is the counts of words in papers, <code>pcounts</code> is the lengths of papers given by 
<code>pcounts = full(sum(wpcounts,1))</code>.
<pre><code>
ww = wpcounts ./ log(1+pcounts(ones(size(wpcounts,1),1),:));
np = length(plist);
h = zeros(1,1000);
for i=1:1000
  pi = 10/np + ww(i,:);
  pi = pi / sum(pi);
  h(i) = -sum(pi.*log(pi));
end
[a i] = sort(h);
wlist = i(1:numw);              % indices of numw most informative words
</code></pre>
</p>

</html>
