hprof
=====

Heap profile reader for Go

Runtime patch: https://codereview.appspot.com/37540043/

You call debug.WriteHeapDump(fd uintptr) to write a heap dump to the given
file descriptor from within your Go program (that's runtime/debug).

The code in this directory is for a hprof utility which converts
from the internal dump format to the hprof format.

go build dumptohprof.go readdump.go
dumptohprof dumpfile dump.hprof
jhat dump.hprof  (might need to download jhat)

then navigate a browser to localhost:7000 and poke around.  A good example is "show heap histogram".

jhat is one simple analysis tool - there are a bunch of others out
there.  My converter only fills in data that jhat requires, though,
other tools may need more info to work.

It's a java-centric format, so there is a lot of junk that doesn't
translate well from Go.

The Go heap contains no type information for objects which have no
pointers in them.  You'll just see "noptrX" types for different sizes
X.

The Go heap also contains no field names for objects, so you'll just
see fX for different offsets X in the object.

Below is a description of the internal format of the heap dump.

The file starts with the bytes "go1.3 heap dump\n".  The rest of the
file is encoded in records of different kind.  Most values in these
records are encoded with a variable-sized integer encoding (uvarint)
compatible with ReadUvarint in encoding/binary.

Each record starts with a "kind" which is uvarint encoded.  These
are the possible kinds:
   1: object
   3: eof
   4: stackroot
   5: dataroot
   6: otherroot
   7: type
   8: stack frame
   9: stack
  10: dump params
  11: finalizer
  12: itab

Strings are encoded with a Uvarint size followed by that many bytes of string (UTF8 encoded)

Each kind of record has a different layout.  Here are the layouts:

object:
  1       uvarint
  addr    uvarint     address of object start
  type    uvarint     address of type, 0 if unknown
  kind    uvarint     0 - regular T, 1 - array of T, 2 - channel of T (must be 0 if type is 0)
  size    uvarint     total size (size of sizeclass, type.size may be a bit smaller)
  data    size bytes

eof:
  3       uvarint

stackroot:
  4       uvarint
  addr    uvarint     address where pointer was found
  rootptr uvarint     possible ptr to an object
  frame   uvarint     sp of frame this pointer came from

dataroot:
  5       uvarint
  addr    uvarint     address where pointer was found
  rootptr uvarint     possible ptr to an object

otherroot:
  6       uvarint
  description string
  rootptr uvarint     possible ptr to an object

type:
  7         uvarint
  addr      uvarint   identifier for type (Type*)
  size      uvarint   size of this type of object
  name      string
  efaceptr  uvarint   1 = the data field of an Eface with this type is a ptr
  nfields   uvarint   number of fields in an object of this type.
  fields    [2]uvarint*  list of <kind,offset> of fields.  Increasing offset order.
                      kinds: 0:Ptr 1:String 2:Slice 3:Iface 4:Eface
  TODO: field names, ...

goroutine:
  8       uvarint
  addr    uvarint     thread identifier
  tos     uvarint     top (the currently running) frame of stack
  goid    uvarint     ID of goroutine
  gopc    uvarint     pc of creation point
  status  uvarint     idle=0, runnable=1, syscall=3, waiting=4
  issystem uvarint
  isbackground uvarint
  waitsince uvarint   
  waitreason string

stack frame:
  9       uvarint
  addr    uvarint     sp of frame
  parent  uvarint     sp of parent's frame, or nil if bottom of stack
  entry   uvarint     pc of function entry
  pc      uvarint     pc code is suspended at
  name    string      function name

dump params:
  10        uvarint
  endian    uvarint     0=little endian, 1=big endian
  ptrsize   uvarint     ptr size in bytes.  4 or 8
  hchansize uvarint     size of channel header in bytes.
  heapstart uvarint     start of heap
  heapend   uvarint     end of heap

finalizer data:
  11        uvarint
  obj       uvarint    object that has a finalizer
  funcval   uvarint    funcval ptr
  code      uvarint    code ptr (funcval->fn)
  fint      uvarint    interface argument type of finalizer function
  ot        uvarint    object type

itab data:
  12        uvarint
  addr      uvarint    Itab*
  ptr       uvarint    1 = the data field of an Iface with this itab is a ptr
