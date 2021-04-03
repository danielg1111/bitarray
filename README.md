bitarray: efficient arrays of booleans
======================================

This module provides an object type which efficiently represents an array
of booleans.  Bitarrays are sequence types and behave very much like usual
lists.  Eight bits are represented by one byte in a contiguous block of
memory.  The user can select between two representations: little-endian
and big-endian.  All of the functionality is implemented in C.
Methods for accessing the machine representation are provided.
This can be useful when bit level access to binary files is required,
such as portable bitmap image files (.pbm).  Also, when dealing with
compressed data which uses variable bit length encoding, you may find
this module useful.


Key features
------------

  * All functionality implemented in C.
  * Bitarray objects behave very much like a list object, in particular
    slicing (including slice assignment and deletion) is supported.
  * The bit endianness can be specified for each bitarray object, see below.
  * Fast methods for encoding and decoding variable bit length prefix codes
  * Bitwise operations: `&`, `|`, `^`, `&=`, `|=`, `^=`, `~`
  * Sequential search
  * Packing and unpacking to other binary data formats, e.g. `numpy.ndarray`.
  * Pickling and unpickling of bitarray objects.
  * Bitarray objects support the buffer protocol
  * `frozenbitarray` objects which are hashable
  * Extensive test suite with about 300 unittests
  * On 32-bit systems, a bitarray object may contain up to 2 Gbits.
  * a separate utility module `bitarray.util`:
      - conversion to hexadecimal string
      - serialization
      - pretty printing
      - conversion to integers
      - creating Huffman codes
      - various count functions
      - other helpful functions


Installation
------------

Bitarray can be installed from source:

    $ tar xzf bitarray-1.9.0.tar.gz
    $ cd bitarray-1.9.0
    $ python setup.py install

On Unix systems, the latter command may have to be executed with root
privileges.  You can also pip install bitarray.  Please note that you need
a working C compiler to run the `python setup.py install` command.
If you rather want to use precompiled binaries, you can:

* `pip install bitarray-hardbyte` (this PyPI package contains Python
  wheels for Linux, MaxOSX and Windows and all common Python versions)
* `conda install bitarray` (both the default Anaconda repository as well
  as conda-forge support bitarray)
* download Windows wheels from
  [Chris Gohlke](https://www.lfd.uci.edu/~gohlke/pythonlibs/#bitarray)

Once you have installed the package, you may want to test it:

    $ python -c 'import bitarray; bitarray.test()'
    bitarray is installed in: /Users/ilan/bitarray/bitarray
    bitarray version: 1.9.0
    sys.version: 2.7.15 (default, Mar  5 2020, 14:58:04) [GCC Clang 9.0.1]
    sys.prefix: /Users/ilan/Mini3/envs/py27
    pointer size: 64 bit
    .........................................................................
    .........................................................................
    ........................................
    ----------------------------------------------------------------------
    Ran 291 tests in 0.638s

    OK

You can always import the function test,
and `test().wasSuccessful()` will return `True` when the test went well.


Using the module
----------------

As mentioned above, bitarray objects behave very much like lists, so
there is not too much to learn.  The biggest difference from list
objects (except that bitarray are obviously homogeneous) is the ability
to access the machine representation of the object.
When doing so, the bit endianness is of importance; this issue is
explained in detail in the section below.  Here, we demonstrate the
basic usage of bitarray objects:

    >>> from bitarray import bitarray
    >>> a = bitarray()            # create empty bitarray
    >>> a.append(True)
    >>> a.extend([False, True, True])
    >>> a
    bitarray('1011')

Bitarray objects can be instantiated in different ways:

    >>> a = bitarray(2**20)       # bitarray of length 1048576 (uninitialized)
    >>> bitarray('1001 011')      # from a string (whitespace is ignored)
    bitarray('1001011')
    >>> lst = [True, False, False, True, False, True, True]
    >>> bitarray(lst)             # from list, tuple, iterable
    bitarray('1001011')

Bits can be assigned from any Python object, if the value can be interpreted
as a truth value.  You can think of this as Python's built-in function bool()
being applied, whenever casting an object:

    >>> a = bitarray([42, '', True, {}, 'foo', None])
    >>> a
    bitarray('101010')
    >>> a.append(a)      # note that bool(a) is True
    >>> a.count(42)      # counts occurrences of True (not 42)
    4
    >>> a.remove('')     # removes first occurrence of False
    >>> a
    bitarray('110101')

Like lists, bitarray objects support slice assignment and deletion:

    >>> a = bitarray(50)
    >>> a.setall(False)
    >>> a[11:37:3] = 9 * bitarray([True])
    >>> a
    bitarray('00000000000100100100100100100100100100000000000000')
    >>> del a[12::3]
    >>> a
    bitarray('0000000000010101010101010101000000000')
    >>> a[-6:] = bitarray('10011')
    >>> a
    bitarray('000000000001010101010101010100010011')
    >>> a += bitarray('000111')
    >>> a[9:]
    bitarray('001010101010101010100010011000111')

In addition, slices can be assigned to booleans, which is easier (and
faster) than assigning to a bitarray in which all values are the same:

    >>> a = 20 * bitarray('0')
    >>> a[1:15:3] = True
    >>> a
    bitarray('01001001001001000000')

This is easier and faster than:

    >>> a = 20 * bitarray('0')
    >>> a[1:15:3] = 5 * bitarray('1')
    >>> a
    bitarray('01001001001001000000')

Note that in the latter we have to create a temporary bitarray whose length
must be known or calculated.


Bitwise operators
-----------------

Bitarray objects support the bitwise operators `&`, `|`, `^`, `<<`, `>>` (as
well as their in-place versions `&=`, `|=`, `^=`, `<<=`, `>>=`.
The behavior is very much what one would expect:

    >>> a = bitarray('101110001')
    >>> b = bitarray('111001011')
    >>> a ^ b
    bitarray('010111010')
    >>> a &= b
    >>> a
    bitarray('101000001')
    >>> a <<= 2
    >>> a
    bitarray('100000100')
    >>> b >> 1
    bitarray('011100101')

The C language does not specify the behavior of negative shifts and
of left shifts larger or equal than the width of the promoted left operand.
The exact behavior is compiler/machine specific.
Our Python bitarray behavior is specified as follows:
  * the length of the bitarray is unchanged by any shift operation
  * blanks are filled by 0
  * negative shifts result in `ValueError: negative shift count`
  * shifts larger or equal to the length of the bitarray result in
    bitarrays with all values 0


Bit endianness
--------------

Since a bitarray allows addressing of individual bits, where the machine
represents 8 bits in one byte, there are two obvious choices for this
mapping: little-endian and big-endian.
When creating a new bitarray object, the endianness can always be
specified explicitly:

    >>> a = bitarray(endian='little')
    >>> a.frombytes(b'A')
    >>> a
    bitarray('10000010')
    >>> b = bitarray('11000010', endian='little')
    >>> b.tobytes()
    b'C'

Here, the low-bit comes first because little-endian means that increasing
numeric significance corresponds to an increasing address (index).
So `a[0]` is the lowest and least significant bit, and `a[7]` is the
highest and most significant bit.

    >>> a = bitarray(endian='big')
    >>> a.frombytes(b'A')
    >>> a
    bitarray('01000001')
    >>> a[6] = 1
    >>> a.tobytes()
    b'C'

Here, the high-bit comes first because big-endian
means "most-significant first".
So `a[0]` is now the lowest and most significant bit, and `a[7]` is the
highest and least significant bit.

The bit endianness is a property attached to each bitarray object.
When comparing bitarray objects, the endianness (and hence the machine
representation) is irrelevant; what matters is the mapping from indices
to bits:

    >>> bitarray('11001', endian='big') == bitarray('11001', endian='little')
    True

Bitwise operations (`&`, `|`, `^`, `&=`, `|=`, `^=`, `~`) are implemented
efficiently using the corresponding byte operations in C, i.e. the operators
act on the machine representation of the bitarray objects.
Therefore, one has to be cautious when applying the operation to bitarrays
with different endianness.

When converting to and from machine representation, using
the `tobytes`, `frombytes`, `tofile` and `fromfile` methods,
the endianness matters:

    >>> a = bitarray(endian='little')
    >>> a.frombytes(b'\x01')
    >>> a
    bitarray('10000000')
    >>> b = bitarray(endian='big')
    >>> b.frombytes(b'\x80')
    >>> b
    bitarray('10000000')
    >>> a == b
    True
    >>> a.tobytes() == b.tobytes()
    False

The endianness can not be changed once an object is created.
However, you can create a new bitarray with different endianness:

    >>> a = bitarray('111000', endian='little')
    >>> b = bitarray(a, endian='big')
    >>> b
    bitarray('111000')
    >>> a == b
    True

The default bit endianness is currently big-endian, however this may change
in the future, and when dealing with the machine representation of bitarray
objects, it is recommended to always explicitly specify the endianness.

Unless explicitly converting to machine representation, using
the `tobytes`, `frombytes`, `tofile` and `fromfile` methods,
the bit endianness will have no effect on any computation, and one
can safely ignore setting the endianness, and other details of this section.


Buffer protocol
---------------

Python 2.7 provides memoryview objects, which allow Python code to access
the internal data of an object that supports the buffer protocol without
copying.  Bitarray objects support this protocol, with the memory being
interpreted as simple bytes.

    >>> a = bitarray('01000001 01000010 01000011', endian='big')
    >>> v = memoryview(a)
    >>> len(v)
    3
    >>> v[-1]
    67
    >>> v[:2].tobytes()
    b'AB'
    >>> v.readonly  # changing a bitarray's memory is also possible
    False
    >>> v[1] = 111
    >>> a
    bitarray('010000010110111101000011')


Variable bit length prefix codes
--------------------------------

The method `encode` takes a dictionary mapping symbols to bitarrays
and an iterable, and extends the bitarray object with the encoded symbols
found while iterating.  For example:

    >>> d = {'H':bitarray('111'), 'e':bitarray('0'),
    ...      'l':bitarray('110'), 'o':bitarray('10')}
    ...
    >>> a = bitarray()
    >>> a.encode(d, 'Hello')
    >>> a
    bitarray('111011011010')

Note that the string `'Hello'` is an iterable, but the symbols are not
limited to characters, in fact any immutable Python object can be a symbol.
Taking the same dictionary, we can apply the `decode` method which will
return a list of the symbols:

    >>> a.decode(d)
    ['H', 'e', 'l', 'l', 'o']
    >>> ''.join(a.decode(d))
    'Hello'

Since symbols are not limited to being characters, it is necessary to return
them as elements of a list, rather than simply returning the joined string.

When the codes are large, and you have many decode calls, most time will
be spent creating the (same) internal decode tree objects.  In this case,
it will be much faster to create a `decodetree` object (which is initialized
with a prefix code dictionary), and can be passed to bitarray's `.decode()`
and `.iterdecode()` methods, instead of passing the prefix code dictionary
to those methods itself.

The above dictionary `d` can be efficiently constructed using the function
`bitarray.util.huffman_code()`.  I also wrote [Huffman coding in Python using
bitarray](http://ilan.schnell-web.net/prog/huffman/) for more background
information.


Reference
=========

The bitarray object:
--------------------

`bitarray(initializer=0, /, endian='big')` -> bitarray

Return a new bitarray object whose items are bits initialized from
the optional initial object, and endianness.
The initializer may be of the following types:

`int`: Create a bitarray of given integer length.  The initial values are
arbitrary.  If you want all values to be set, use the .setall() method.

`str`: Create bitarray from a string of `0` and `1`.

`list`, `tuple`, `iterable`: Create bitarray from a sequence, each
element in the sequence is converted to a bit using its truth value.

`bitarray`: Create bitarray from another bitarray.  This is done by
copying the buffer holding the bitarray data, and is hence very fast.

The optional keyword arguments `endian` specifies the bit endianness of the
created bitarray object.
Allowed values are the strings `big` and `little` (default is `big`).

Note that setting the bit endianness only has an effect when accessing the
machine representation of the bitarray, i.e. when using the methods: tofile,
fromfile, tobytes, frombytes.


**A bitarray object supports the following methods:**

`all()` -> bool

Return True when all bits in the array are True.


`any()` -> bool

Return True when any bit in the array is True.


`append(item, /)`

Append the truth value `bool(item)` to the end of the bitarray.


`buffer_info()` -> tuple

Return a tuple (address, size, endianness, unused, allocated) giving the
memory address of the bitarray's buffer, the buffer size (in bytes),
the bit endianness as a string, the number of unused bits within the last
byte, and the allocated memory for the buffer (in bytes).


`bytereverse()`

For all bytes representing the bitarray, reverse the bit order (in-place).
Note: This method changes the actual machine values representing the
bitarray; it does not change the endianness of the bitarray object.


`clear()`

Remove all items from the bitarray.


`copy()` -> bitarray

Return a copy of the bitarray.


`count(value=True, start=0, stop=<end of array>, /)` -> int

Count the number of occurrences of bool(value) in the bitarray.


`decode(code, /)` -> list

Given a prefix code (a dict mapping symbols to bitarrays, or `decodetree`
object), decode the content of the bitarray and return it as a list of
symbols.


`encode(code, iterable, /)`

Given a prefix code (a dict mapping symbols to bitarrays),
iterate over the iterable object with symbols, and extend the bitarray
with the corresponding bitarray for each symbol.


`endian()` -> str

Return the bit endianness of the bitarray as a string (`little` or `big`).


`extend(iterable, /)`

Extend bitarray by appending the truth value of each element given
by iterable.  If a string is provided, each `0` and `1` are appended
as bits (whitespace is ignored).


`fill()` -> int

Add zeros to the end of the bitarray, such that the length of the bitarray
will be a multiple of 8, and return the number of bits added (0..7).


`frombytes(bytes, /)`

Extend bitarray with raw bytes.  That is, each append byte will add eight
bits to the bitarray.


`fromfile(f, n=-1, /)`

Extend bitarray with up to n bytes read from the file object f.
When n is omitted or negative, reads all data until EOF.
When n is provided and positive but exceeds the data available,
EOFError is raised (but the available data is still read and appended.


`index(value, start=0, stop=<end of array>, /)` -> int

Return index of the first occurrence of `bool(value)` in the bitarray.
Raises `ValueError` if the value is not present.


`insert(index, value, /)`

Insert `bool(value)` into the bitarray before index.


`invert(index=<all bits>, /)`

Invert all bits in the array (in-place).
When the optional `index` is given, only invert the single bit at index.


`iterdecode(code, /)` -> iterator

Given a prefix code (a dict mapping symbols to bitarrays, or `decodetree`
object), decode the content of the bitarray and return an iterator over
the symbols.


`itersearch(bitarray, /)` -> iterator

Searches for the given a bitarray in self, and return an iterator over
the start positions where bitarray matches self.


`length()` -> int

Return the length - a.length() is the same as len(a).
Deprecated since 1.5.1, use len().


`pack(bytes, /)`

Extend the bitarray from bytes, where each byte corresponds to a single
bit.  The byte `b'\x00'` maps to bit 0 and all other characters map to
bit 1.
This method, as well as the unpack method, are meant for efficient
transfer of data between bitarray objects to other python objects
(for example NumPy's ndarray object) which have a different memory view.


`pop(index=-1, /)` -> item

Return the i-th (default last) element and delete it from the bitarray.
Raises `IndexError` if bitarray is empty or index is out of range.


`remove(value, /)`

Remove the first occurrence of `bool(value)` in the bitarray.
Raises `ValueError` if item is not present.


`reverse()`

Reverse the order of bits in the array (in-place).


`search(bitarray, limit=<none>, /)` -> list

Searches for the given bitarray in self, and return the list of start
positions.
The optional argument limits the number of search results to the integer
specified.  By default, all search results are returned.


`setall(value, /)`

Set all bits in the bitarray to `bool(value)`.


`sort(reverse=False)`

Sort the bits in the array (in-place).


`to01()` -> str

Return a string containing '0's and '1's, representing the bits in the
bitarray object.


`tobytes()` -> bytes

Return the byte representation of the bitarray.
When the length of the bitarray is not a multiple of 8, the few remaining
bits (1..7) are considered to be 0.


`tofile(f, /)`

Write the byte representation of the bitarray to the file object f.
When the length of the bitarray is not a multiple of 8,
the remaining bits (1..7) are set to 0.


`tolist(as_ints=False, /)` -> list

Return a list with the items (False or True) in the bitarray.
The optional parameter, changes the items in the list to integers (0 or 1).
Note that the list object being created will require 32 or 64 times more
memory (depending on the machine architecture) than the bitarray object,
which may cause a memory error if the bitarray is very large.


`unpack(zero=b'\x00', one=b'\xff')` -> bytes

Return bytes containing one character for each bit in the bitarray,
using the specified mapping.


The frozenbitarray object:
--------------------------

This object is very similar to the bitarray object.  The difference is that
this a frozenbitarray is immutable, and hashable:

    >>> from bitarray import frozenbitarray
    >>> a = frozenbitarray('1100011')
    >>> a[3] = 1
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
      File "bitarray/__init__.py", line 40, in __delitem__
        raise TypeError("'frozenbitarray' is immutable")
    TypeError: 'frozenbitarray' is immutable
    >>> {a: 'some value'}
    {frozenbitarray('1100011'): 'some value'}

`frozenbitarray(initializer=0, /, endian='big')` -> frozenbitarray

Return a frozenbitarray object, which is initialized the same way a bitarray
object is initialized.  A frozenbitarray is immutable and hashable.
Its contents cannot be altered after it is created; however, it can be used
as a dictionary key.


The decodetree object:
----------------------

This (immutable and unhashable) object stores a binary tree initialized
from a prefix code dictionary.  It's sole purpose is to be passed to
bitarray's `.decode()` and `.iterdecode()` methods, instead of passing
the prefix code dictionary to those methods directly:

    >>> from bitarray import bitarray, decodetree
    >>> t = decodetree({'a': bitarray('0'), 'b': bitarray('1')})
    >>> a = bitarray('0110')
    >>> a.decode(t)
    ['a', 'b', 'b', 'a']
    >>> ''.join(a.iterdecode(t))
    'abba'

`decodetree(code, /)` -> decodetree

Given a prefix code (a dict mapping symbols to bitarrays),
create a binary tree object to be passed to `.decode()` or `.iterdecode()`.


Functions defined in the `bitarray` module:
--------------------------------------------

`bits2bytes(n, /)` -> int

Return the number of bytes necessary to store n bits.


`get_default_endian()` -> string

Return the default endianness for new bitarray objects being created.
Under normal circumstances, the return value is `big`.


`test(verbosity=1, repeat=1)` -> TextTestResult

Run self-test, and return unittest.runner.TextTestResult object.


Functions defined in `bitarray.util` module:
--------------------------------------------

`zeros(length, /, endian=None)` -> bitarray

Create a bitarray of length, with all values 0, and optional
endianness, which may be 'big', 'little'.


`urandom(length, /, endian=None)` -> bitarray

Return a bitarray of `length` random bits (uses `os.urandom`).


`pprint(bitarray, /, stream=None, group=8, indent=4, width=80)`

Prints the formatted representation of object on `stream`, followed by a
newline.  If `stream` is `None`, `sys.stdout` is used.  By default, elements
are grouped in bytes (8 elements), and 8 bytes (64 elements) per line.
Non-bitarray objects are printed by the standard library
function `pprint.pprint()`.


`make_endian(bitarray, endian, /)` -> bitarray

When the endianness of the given bitarray is different from `endian`,
return a new bitarray, with endianness `endian` and the same elements
as the original bitarray.
Otherwise (endianness is already `endian`) the original bitarray is returned
unchanged.


`rindex(bitarray, value=True, /)` -> int

Return the rightmost index of `bool(value)` in bitarray.
Raises `ValueError` if the value is not present.


`strip(bitarray, mode='right', /)` -> bitarray

Return a new bitarray with zeros stripped from left, right or both ends.
Allowed values for mode are the strings: `left`, `right`, `both`


`count_n(a, n, /)` -> int

Return the smallest index `i` for which `a[:i].count() == n`.
Raises `ValueError`, when n exceeds total count (`a.count()`).


`parity(a, /)` -> bool

Return the parity of bitarray `a`.  This is equivalent
to `bool(a.count() % 2)` (but more efficient).


`count_and(a, b, /)` -> int

Return `(a & b).count()` in a memory efficient manner,
as no intermediate bitarray object gets created.


`count_or(a, b, /)` -> int

Return `(a | b).count()` in a memory efficient manner,
as no intermediate bitarray object gets created.


`count_xor(a, b, /)` -> int

Return `(a ^ b).count()` in a memory efficient manner,
as no intermediate bitarray object gets created.


`subset(a, b, /)` -> bool

Return True if bitarray `a` is a subset of bitarray `b` (False otherwise).
`subset(a, b)` is equivalent to `(a & b).count() == a.count()` but is more
efficient since we can stop as soon as one mismatch is found, and no
intermediate bitarray object gets created.


`ba2hex(bitarray, /)` -> hexstr

Return a string containing the hexadecimal representation of
the bitarray (which has to be multiple of 4 in length).


`hex2ba(hexstr, /, endian=None)` -> bitarray

Bitarray of hexadecimal representation.  hexstr may contain any number
(including odd numbers) of hex digits (upper or lower case).


`ba2base(n, bitarray, /)` -> str

Return a string containing the base `n` ASCII representation of
the bitarray.  Allowed values for `n` are 2, 4, 8, 16, 32 and 64.
The bitarray has to be multiple of length 1, 2, 3, 4, 5 or 6 respectively.
For `n=16` (hexadecimal), `ba2hex()` will be much faster, as `ba2base()`
does not take advantage of byte level operations.
For `n=32` the RFC 4648 Base32 alphabet is used, and for `n=64` the
standard base 64 alphabet is used.


`base2ba(n, asciistr, /, endian=None)` -> bitarray

Bitarray of the base `n` ASCII representation.
Allowed values for `n` are 2, 4, 8, 16 and 32.
For `n=16` (hexadecimal), `hex2ba()` will be much faster, as `base2ba()`
does not take advantage of byte level operations.
For `n=32` the RFC 4648 Base32 alphabet is used, and for `n=64` the
standard base 64 alphabet is used.


`ba2int(bitarray, /, signed=False)` -> int

Convert the given bitarray into an integer.
The bit-endianness of the bitarray is respected.
`signed` indicates whether two's complement is used to represent the integer.


`int2ba(int, /, length=None, endian=None, signed=False)` -> bitarray

Convert the given integer to a bitarray (with given endianness,
and no leading (big-endian) / trailing (little-endian) zeros), unless
the `length` of the bitarray is provided.  An `OverflowError` is raised
if the integer is not representable with the given number of bits.
`signed` determines whether two's complement is used to represent the integer,
and requires `length` to be provided.
If signed is False and a negative integer is given, an OverflowError
is raised.


`serialize(bitarray, /)` -> bytes

Return a serialized representation of the bitarray, which may be passed to
`deserialize()`.  It efficiently represents the bitarray object (including
its endianness) and is guaranteed not to change in future releases.


`deserialize(bytes, /)` -> bitarray

Return a bitarray given the bytes representation returned by `serialize()`.


`huffman_code(dict, /, endian=None)` -> dict

Given a frequency map, a dictionary mapping symbols to their frequency,
calculate the Huffman code, i.e. a dict mapping those symbols to
bitarrays (with given endianness).  Note that the symbols may be any
hashable object (including `None`).


Change log
----------

2021-04-XX   1.9.0:

  * add `bitarray.util.ba2base()` and `bitarray.util.base2ba()`,
    see last paragraph in [Bitarray representations](examples/represent.md)
  * documentation and tests


*1.8.2* (2021-03-31):

  * fix crash caused by unsupported types in binary operations, #116
  * speedup initializing or extending a bitarray from another with different
    bit endianness
  * add formatting options to `bitarray.util.pprint()`
  * add documentation on [bitarray representations](examples/represent.md)
  * add and improve tests (all 291 tests run in less than half a second on
    a modern machine)


*1.8.1* (2021-03-25):

  * moved implementation of and `hex2ba()` and `ba2hex()` to C-level
  * add `bitarray.util.parity()`


*1.8.0* (2021-03-21):

  * add `bitarray.util.serialize()` and `bitarray.util.deserialize()`
  * allow whitespace (ignore space and `\n\r\t\v`) in input strings,
    e.g. `bitarray('01 11')` or `a += '10 00'`
  * add `bitarray.util.pprint()`
  * When initializing a bitarray from another with different bit endianness,
    e.g. `a = bitarray('110', 'little')` and `b = bitarray(a, 'big')`,
    the buffer used to be simply copied, with consequence that `a == b` would
    result in `False`.  This is fixed now, that is `a == b` will always
    evaluate to `True`.
  * add example showing how to [jsonize bitarrays](examples/extend_json.py)
  * add tests


*1.7.1* (2021-03-12):

  * fix issue #114, raise TypeError when incorrect index is used during
    assignment, e.g. `a[1.5] = 1`
  * raise TypeError (not IndexError) when assigning slice to incorrect type,
    e.g. `a[1:4] = 1.2`
  * improve some docstrings and tests


*1.7.0* (2021-02-27):

  * add `bitarray.util.urandom()`
  * raise TypeError when trying to extend bitarrays from bytes on Python 3,
    ie. `bitarray(b'011')` and `.extend(b'110')`.  (Deprecated since 1.4.1)


*1.6.3* (2021-01-20):

  * add missing .h files to sdist tarball, #113


*1.6.2* (2021-01-20):

  * use `Py_SET_TYPE()` and `Py_SET_SIZE()` for Python 3.10, #109
  * add official Python 3.10 support
  * fix slice assignment to same object,
    e.g. `a[2::] = a` or `a[::-1] = a`, #112
  * add bitarray.h, #110


*1.6.1* (2020-11-05):

  * use PyType_Ready for all types: bitarray, bitarrayiterator,
    decodeiterator, decodetree, searchiterator


*1.6.0* (2020-10-17):

  * add `decodetree` object, for speeding up consecutive calls
    to `.decode()` and `.iterdecode()`, in particular when dealing
    with large prefix codes, see #103
  * add optional parameter to `.tolist()` which changes the items in the
    returned list to integers (0 or 1), as opposed to Booleans
  * remove deprecated `bitdiff()`, which has been deprecated since version
    1.2.0, use `bitarray.util.count_xor()` instead
  * drop Python 2.6 support
  * update license file, #104


*1.5.3* (2020-08-24):

  * add optional index parameter to `.index()` to invert single bit
  * fix `sys.getsizeof(bitarray)` by adding `.__sizeof__()`, see issue #100


*1.5.2* (2020-08-16):

  * add PyType_Ready usage, issue #66
  * speedup search() for bitarrays with length 1 in sparse bitarrays,
    see issue #67
  * add tests


*1.5.1* (2020-08-10):

  * support signed integers in `util.ba2int()` and `util.int2ba()`,
    see issue #85
  * deprecate `.length()` in favor of `len()`


*1.5.0* (2020-08-05):

  * Use `Py_ssize_t` for bitarray index.  This means that on 32bit
    systems, the maximum number of elements in a bitarray is 2 GBits.
    We used to have a special 64bit index type for all architectures, but
    this prevented us from using Python's sequence, mapping and number
    methods, and made those method lookups slow.
  * speedup slice operations when step size = 1 (if alignment allows
    copying whole bytes)
  * Require equal endianness for operations: `&`, `|`, `^`, `&=`, `|=`, `^=`.
    This should have always been the case but was overlooked in the past.
  * raise TypeError when trying to create bitarray from boolean
  * This will be last release to still support Python 2.6 (which was retired
    in 2013).  We do NOT plan to stop support for Python 2.7 anytime soon.


Please find the complete change log [here](https://github.com/ilanschnell/bitarray/blob/master/CHANGE_LOG).
