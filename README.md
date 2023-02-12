# jzon

A correct and safe(er) JSON [RFC 8259][JSONRFC] parser with sane defaults.

Please see the section [Motivation and Features](#motivation-and-features) for a set of motivations driving jzon and why you should consider it over the other hundred options available for JSON in CL.

[![Actions Status](https://github.com/Zulu-Inuoe/jzon/workflows/ci/badge.svg)](https://github.com/Zulu-Inuoe/jzon/actions)

#### Table of Contents

* [Quickstart](#quickstart) 
  * [Reading](#reading)
  * [Writing](#writing)
  * [Type Mappings](#type-mappings)
* [Usage](#usage)
  * [`jzon:parse`](#jzonparse)
  * [`jzon:stringify`](#jzonstringify)
    * [Additional Types for Writing](#additionally-supported-types-for-writing)
  * [`jzon:writer`](#jzonwriter)
    * [`jzon:make-writer`](#jzonmakewriter)
    * [`jzon:with-writer*`](#jzonwith-writer)
    * [`jzon:write-value`](#jzonwrite-value)
    * [Other Streaming Writer Functions](#other-streaming-writer-functions)
    * [Streaming Writer Example](#streaming-writer-example)
  * [Custom Serialization](#custom-serialization)
    * [standard-object](#standard-object)
    * [Specializing `jzon:coerced-fields`](#specializing-coerced-fields)
    * [Specializing `jzon:write-value`](#specializing-jzonwrite-value)
  * [`jzon:parser`](#jzon:parser)
    * [`jzon:with-parser`](#jzonwith-parser)
    * [`jzon:parse-next`](#jzonparse-next)
    * [Streaming Parser Example](#streaming-parser-example)

* [Motivation and Features](#motivation-and-features)
  * [Safety](#safety)
  * [Correctness](#correctness)
  * [Simplicity](#simplicity)
  * [Performance](#performance)
* [Dependencies](#dependencies)
* [License](#license)
* [Alternatives](#alternatives)

# Quickstart

**Note**: *Examples in this README can be copy-pasted in your REPL assuming you've got a nickname set up for jzon. Try `(add-package-local-nickname '#:jzon '#:com.inuoe.jzon)` while in CL-USER.*

Most users will simply use `jzon:parse` for reading, and `jzon:stringify` for writing. These mirror the [JSON methods in JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON).

### Reading

`jzon:parse` will parse JSON and produce a CL value

```lisp
(jzon:parse "{
  \"license\": null,
  \"active\": false,
  \"important\": true,
  \"id\": 1,
  \"xp\": 3.2,
  \"name\": \"Rock\",
  \"tags\":  [
    \"alone\"
  ]
}") 
#| => #<HASH-TABLE :TEST EQUAL :COUNT 7 {100693CD63} |#

(defparameter *ht* *)

(equalp 'null       (gethash "licence" *ht*))
(equalp nil         (gethash "active" *ht*))
(equalp t           (gethash "important" *ht*))
(equalp 1           (gethash "id" *ht*))
(equalp 3.2d0       (gethash "xp" *ht*))
(equalp "Rock"      (gethash "name" *ht*))
(equalp #("alone")  (gethash "tags" *ht*))
```

### Writing

`jzon:stringify` will serialize a value to JSON:

```lisp
(jzon:stringify #(null nil t 42 3.14 "Hello, world!") :stream t :pretty t)
```
```json
[
  null,
  false,
  true,
  42,
  3.14,
  "Hello, world!"
 ]
```

### Type Mappings

jzon cannonically maps types per the following chart:

| JSON   | CL                      |
|--------|-------------------------|
| true   | symbol `t`              |
| false  | symbol `nil`            |
| null   | symbol `null`           |
| number | integer or double-float |
| string | simple-string           |
| array  | simple-vector           |
| object | hash-table (equal)      |

**Note** the usage of symbol `cl:null` as a sentinel for JSON `null`

When writing, additional values are supported. Please see the section [jzon:stringify](#writing).

# Usage

As noted, `jzon:parse` and `jzon:stringify` suit most use-cases, this section goes into more detail, as well as an introduction to the [`jzon:writer` interface](#jzonwriter).

### jzon:parse

*Function* **jzon:parse** *in &key max-depth allow-comments allow-trailing-comma max-string-length key-fn*

*=> value* 

* *in* - a `string`, `vector (unsigned-byte 8)`, `stream`, or `pathname`
* *max-depth* - a positive `integer`, or `nil`
* *allow-comments* - a `boolean`
* *allow-trailing-comma* - a `boolean`
* *max-string-length* - a positive `integer`
* *key-fn* - a designator for a function of one argument, or `nil`

*value* - a `jzon:json-element` (see [Type Mappings](#type-mappings))

#### Description

Reads JSON from `in` and returns a `jzon:json-element` per [Type Mappings](#type-mappings).

`in` can be any of the following types:

* string
* (vector (unsigned-byte 8)) - octets in utf-8
* stream - character or binary in utf-8
* pathname - `jzon:parse` will open the file for reading in utf-8

The keyword arguments control optional features when reading:
* `:allow-comments` controls if we allow single-line // comments and /**/ multiline block comments.
* `:allow-trailing-comma` controls if we allow a single comma `,` after all elements of an array or object.
* `:key-fn` is a function of one value which is called on object keys as they are read, or null *(see below)*
* `:max-depth` controls the maximum depth allowed when nesting arrays or objects.
* `:max-string-length` controls the maximum length allowed when reading a string key or value.

##### *key-fn*

When parsing objects, *key-fn* is called on each of that object's keys (`simple-string`):

```lisp
(jzon:parse "{ \"x\": 0, \"y\": 1 }" :key-fn #'print)
"x"
"y"
#| #<HASH-TABLE :TEST EQUAL :COUNT 2 {1006942E83}> |#
```

the role of *key-fn* is to allow the user to control how keys end up as hash table keys. The default *key-fn* will share object keys between all objects during parse. See [Object Key Pooling](#object-key-pooling).

As an example, `alexandria:make-keyword` can be used to make object keys into keywords:

```lisp
(jzon:parse "[ { \"x\": 1, \"y\": 2 }, { \"x\": 3, \"y\": 4 } ]" :key-fn #'alexandria:make-keyword)

(defparameter *v* *)

(gethash :|x| (aref *v* 0)) #| => 1 |#
(gethash :|y| (aref *v* 0)) #| => 2 |#
```

Another example is to use `#'identity` in order to avoid [key pooling](#object-key-pooling):

```lisp
(jzon:parse "[ { \"x\": 1, \"y\": 2 }, { \"x\": 3, \"y\": 4 } ]" :key-fn #'identity)

(defparameter *v* *)

(gethash "x" (aref *v* 0)) #| => 1 |#
(gethash "y" (aref *v* 0)) #| => 2 |#
```

This *may* help speed up parsing on highly heterogeneous JSON.

**Note**: It is recommended leave this as default. The performance improvement is usually not substantive enough to warrant duplicated strings, and interning strings from untrusted JSON is a security risk.

### jzon:stringify

*Function* **jzon:stringify** *value* *&key stream pretty coerce-key replacer*

*=> result*

* *value* - a `jzon:json-element`, or other value (see below)
* *stream* - a destination like in `format`, or a `pathname`
* *pretty* - a boolean
* *coerce-key* - a function of one argument, or nil (see below)
* *replacer* - a function of two arguments (see below)

*result* - nil, or a string

#### Description

Serializes *value* to JSON and writes it to *stream*.

If *pretty* is true, the output is formatted with spaces and newlines. 

In addition to serializing `json:jzon-element` values per [Type Mappings](#type-mappings), `jzon:stringify` allows other values. 
See [Additionally Supported Types For Writing](#additionally-supported-types-for-writing) and [Custom Serialization](#custom-serialization).

##### stream

*stream* is a destination as in `format`, or a `pathname`:

* `t` - Writes to `*standard-output*`
* `nil` - Writes to a fresh string and returns it
* an open `stream` - Writes to that stream
* a `string` with a fill-pointer - writes to that string as `with-output-to-string`
* a `pathname` - Must designate a file. Creates or supersedes a new file and writes to it

##### coerce-key

A function for coercing keys to strings. See [Custom Serialization](#custom-serialization).

##### replacer

A designator for a 'replacer' function as outlined in the JavaScript [JSON.parse](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify) method.

However, due to the lack of `undefined` in CL, we make use of multiple return values to indicate replacement.

```lisp
(jzon:stringify #("first" "second" "third")
                :stream t :pretty t
                :replacer (lambda (key value)
                            (case key
                              #| Always include toplevel |#
                              ((nil) t)
                              #| Do not include |#
                              (0 nil)
                              #| Include |#
                              (1 t)
                              #| Include, but replace |#
                              (2 (values t (format nil "Lupin the ~A" value))))))
```
```json
[
  "second",
  "Lupin the third"
]
```

#### Additionally Supported Types For Writing

When writing, the following type mappings are also available:

| CL                  | JSON                                                                |
|---------------------|---------------------------------------------------------------------|
| symbol              | string (`symbol-name`, but see [Symbol key case](#symbol-key-case)) |
| character           | string (`string`)                                                   |
| pathname            | string (`uiop:native-namestring`)                                   |
| real                | number                                                              |
| sequence            | array                                                               |
| standard-object     | object                                                              |
| structure-object\*  | object                                                              |

\*: On supported implementations where structure slots are available via the MOP.

If you have an alist/plist you wish to write, we recommend the use of either `alexandria:alist-hash-table` or `alexandria:plist-hash-table`, or use one of the methods in [Custom Serialization](#custom-serialization).

Previously, jzon attempted to detect alists/plists, but this was error-prone and came with many edge-cases.

##### Symbol key case

By default, when a symbol represents a **key** in a JSON object, its name will be downcased, unless it contains mixed-case characters.

For example:

```lisp
(let ((ht (make-hash-table :test 'equal)))
  (setf (gethash 'only-keys ht) 'are-affected)
  (setf (gethash '|noChange| ht) '|when used|)
  (setf (gethash "AS A" ht) '|value|)

  (jzon:stringify ht :pretty t :stream t))
```
``` json
{
  "only-keys": "ARE-AFFECTED",
  "noChange": "when Used",
  "AS A": "value"
}
```

Please see [Custom Serialization](#custom-serialization) and [write-value](#write-value) for more details.

### jzon:writer

A second way of writing JSON is to use the `jzon:writer` API, which allows you to fully control the values, order, and types, including arbitrary logic.

An example to start:

```lisp
(jzon:with-writer* (:stream *standard-output* :pretty t)
  (jzon:with-object*
    (jzon:write-key* :age)
    (jzon:write-value* 24)
    
    (jzon:write-property :colour :blue)
    
    (jzon:write-properties* :outside nil
                            :interests #()
                            :talent 'null)

    (jzon:write-key* "an-array")
    (jzon:with-array*
      (jzon:write-values* :these :are :elements))

    (jzon:write-key* "another array")
    (jzon:write-array* :or "you" "can use this")))
```
```json
{
  "age": 24,
  "colour": "BLUE",
  "outside": false,
  "interests": [],
  "talens": null,
  "an-array": [
    "THESE",
    "ARE",
    "ELEMENTS"
  ],
  "another array": [
    "OR",
    "you",
    "can use this"
  ]
}
```

`jzon:make-writer` and `jzon:with-writer*` accept the same arguments as `jzon:stringify`, *except* `:stream` must be an open `cl:stream`.

**Note** All writer-related functions are duplicated in ones suffixed with `*` which use the `jzon:*writer*` special variable, and ones lacking the suffix, where the writer is the first argument.

For example, these two are equivalent:
```lisp
(jzon:with-writer* ()
  (write-value* "foo"))
```
```lisp
(with-writer (writer)
  (write-value writer "foo"))
```

### jzon:make-writer

*Function* **jzon:make-writer** *&key stream pretty coerce-key replacer => writer*

* *stream* - an open character or binary output `stream`
* *pretty* - a boolean
* *coerce-key* - a function of one argument, or nil (see below)
* *replacer* - a function of two arguments (see below)

*writer* - a `jzon:writer`

#### Description

Construct a writer for writing JSON in a streaming fashion.

Generally akin to [`jzon:stringify`](#jzonstringify), with the exception that *stream* must be an already open `stream`.

If *pretty* is true, all output is indented with spaces and newlines.

##### coerce-key

A function for coercing keys to strings. See [Custom Serialization](#custom-serialization).

##### replacer

Please see the section in [`jzon:stringify`](#jzonstringify).

### jzon:with-writer*

*Macro* **jzon:with-writer** *(&rest args) declaration\* form\**

* *args* - initialization arguments to `jzon:make-writer`
* *declaration* - a declare expression, not evaluated
* *forms* - an implicig progn

#### Description

Create a new writer and bind it locally to `jzon:*writer*`

Essentially equivalent to

```lisp
(let ((jzon:*writer* (jzon:make-writer args)))
  form)
```

### jzon:write-value

*Generic Function* **jzon:write-value** *writer* *value*

* *writer* - a `jzon:writer`
* *value* - a `jzon:json-element`, or other value (see below) 

*=> writer*

#### Description

`jzon:write-value writer value` - Writes any `value` to the `writer`. Usable when writing a toplevel value, array element, or object property value.

```lisp
(jzon:write-value writer "Hello, world")
```

`value` can be any `jzon:json-element`, but other values supported. See [Custom Serialization](#custom-serialization).

### Other Streaming Writer Functions

Here we briefly document all the additional helper functions for interfacing with the `jzon:writer`.

Because the entire API is duplicated, we only refer to the `*`-suffixed functions here for brevity.

**Note:** *To save in verbosity in the following examples, we assume to have a `jzon:*writer*` bound in the following examples.*

*For trying at the REPL, use something like:*
```lisp
#| Bind `jzon:*writer*` |#
(setf jzon:*writer* (jzon:make-writer :stream *standard-output* :pretty t))

#| Start an array so we can write multiple values |#
(jzon:begin-array*)
```

##### Arrays

`jzon:begin-array writer` - Begin writing an array

`json:end-array writer` - Finish writing an array.

```lisp
(jzon:begin-array*)
(jzon:write-value* 0)
(jzon:write-value* 1)
(jzon:write-value* 2)
(jzon:end-array*)
```

`jzon:with-array writer` - Open a block to begin writing array values.

```lisp
(jzon:with-array*
  (jzon:write-value* 0)
  (jzon:write-value* 1)
  (jzon:write-value* 2))
```

`jzon:write-values writer &rest values*` - Write several array values.
```lisp
(jzon:with-array*
  (jzon:write-values* 0 1 2))
```


`jzon:write-array` - Open a new array, write its values, and close it.

```lisp
(jzon:write-array* 0 1 2)
```

##### Objects 

*Function* `jzon:begin-object writer` - Begin writing an object.

*Function* `json:end-object writer` - Finish writing an object.

```lisp
(jzon:begin-object*)
(jzon:write-property* "age" 42)
(jzon:end-object*)
```

*Macro* `jzon:with-object writer` - Open a block where you can begin writing object properties.

```lisp
(jzon:with-object*
  (jzon:write-property* "age" 42))
```

*Function* `jzon:write-key writer key` - Write an object key.

```lisp
(jzon:with-object*
  (jzon:write-key* "age")
  (jzon:write-value* 42))
```

*Function* `json:write-property writer key value` - Write an object key and value.

```lisp
(jzon:with-object*
  (jzon:write-property* "age" 42))
```

*Function* `jzon:write-properties writer &rest key* value*` - Write several object keys and values.

```lisp
(jzon:with-object*
  (jzon:write-properties* "age" 42
                          "colour" "blue"
                          "x" 0
                          "y" 10))
```

*Function* `jzon:write-object writer &rest key* value*` - Open a new object, write its keys and values, and close it.

```lisp
(jzon:write-object* "age" 42
                    "colour" "blue"
                    "x" 0
                    "y" 10)
```


### Streaming Writer Example

`jzon:stringify` could be approximately defined as follows:

```lisp
(defun my/jzon-stringify (value)
  (labels ((recurse (value)
             (etypecase value
               (jzon:json-atom
                 (jzon:write-value* value))
               (vector
                 (jzon:with-array*
                   (map nil #'recurse value)))
               (hash-table
                 (jzon:with-object*
                   (maphash (lambda (k v)
                              (jzon:write-key* k)
                              (recurse v))
                            value))))))
    (with-output-to-string (s)
      (jzon:with-writer* (:stream s)
        (recurse value)))))
```

## Custom Serialization

When using either [`jzon:stringify`](#jzonstringify) or [`jzon:write-value`](#jzonwrite-value), you can customize writing of any values not covered in the [Type Mappings](#type-mappings) in an few different ways.

The call graph looks like this:

`jzon:write-value` => `(method standard-object)` => `jzon:coerced-fields`

### standard-object

By default, if your object is a `standard-object`, it will be serialized as a JSON object, using each of its **bound** slots as keys.

A slot's `:type` is used to interpret the meaning of `nil` in that slot:

1. `boolean` - `false`
2. `list` - `[]`
3. `null` - `null`

**Note**: When unspecified, a slot will serialize `nil` as `null`.

#### standard-object Serialization Example

Consider the following classes:

```lisp
(defclass job ()
  ((company
    :initarg :company
    :reader company)
   (title
    :initarg :title
    :reader title)))

(defclass person ()
  ((name
     :initarg :name
     :reader name)
   (alias
     :initarg :alias)
   (job
     :initarg :job
     :reader job)
   (married
     :initarg :married
     :type boolean)
   (children
    :initarg :children
    :type list)))
```

Now consider the following scenarios:

```lisp
(jzon:stringify (make-instance 'person :name "Anya" :job nil
                               :married nil :children nil)
                :pretty t :stream t)`
```
``` json
{
  "name": "Anya",
  "job": null,
  "married": false,
  "children": []
}
```

1. `alias` is omitted, because it is unbound
2. `job` serializes as `null`, because it has no specified `:type`
3. `married` serializes as `false`, because it is specified as a `:boolean`
4. `children` serializes as `[]`, because it is specified as a `list`

A second example:
```lisp

(jzon:stringify (make-instance 'person :name "Loid" :alias "Twilight" 
                               :job (make-instance 'job :company "WISE" :title "Agent")
                               :married t
                               :children (list (make-instance 'person :name "Anya"
                                                              :job nil :married nil 
                                                              :children nil)))
                :pretty t :stream t)
```
```json
{
  "name": "Loid",
  "alias": "Twilight",
  "job": {
    "company": "WISE",
    "title": "Agent"
  },
  "married": true,
  "children": [
    {
      "name": "Anya",
      "job": null,
      "married": false,
      "children": []
    }
  ]
}
```

Here we can note:

1. We now include `alias` as it is bound
2. `job` recurses into the `job` object
3. `married` is t, which serializes as `true`
4. `children` now contains a child element

If you require more control, please see the generic function [`jzon:coerced-fields`](#coerced-fields).

### Specializing coerced-fields

The generic function `jzon:coerced-fields` is called by jzon when writing a value as a JSON object in order to find what properties that object has.

It is useful when the [standard-object](#standard-object) almost does what you want, but you want some more control.

`jzon:coerced-fields` should return a list of 'fields', which are two (or three) element lists of the form:

```lisp
(name value &optional type)
```

* `name` can be any suitable key name.
* `value` can be any value - it'll be coerced if necessary.
* `type` is used as `:type` above, in order to resolve ambiguities with `nil`.

#### coerced-fields Example

Consider our previous `person` class. Say we wish to:

1. Show their `name`
2. Add a `type` to specify they are a person
3. Show `false` for their job if not applicable 

``` lisp
(defmethod jzon:coerced-fields ((person person))
  (list (list "name" (name person))
        (list "type" "person")
        (list "job" (job person) 'boolean)))
```

now

```lisp
(jzon:stringify (make-instance 'person :name "Anya" :job nil
                               :married nil :children nil) 
                :pretty t :stream t)`
```
``` json
{
  "name": "Anya",
  "type": "person",
  "job": false
}
```

If you require even more control, please see the section on (write-values)[#write-values] where we make use of the [writer](#streaming-writer) API.

### Specializing jzon:write-value

The final way to support custom serialization, is the `jzon:write-value` generic function.

This allows you to emit whatever values you wish for a given object.

Once again considering our `person` and `job` classes above, we can specialize a method for `jzon:write-value` on `job`:

```lisp
(defmethod jzon:write-value (writer (job job))
  (cond
    ((string= (company job) "WISE")
      (jzon:write-object writer
                         "company" "Eastern Healthcare"
                         "title" (aref #("Psychologist" "Physician" "Janitor" "Surgeon" "Receptionist") (random 5))))
    ((string= (title job) "Assassin")
      (jzon:with-object writer
        (jzon:write-properties writer
                               "company" "City Hall"
                               "title" "Clerk")
        (jzon:write-key writer "lifelines")
        (jzon:write-array writer "Yuri" "Camilla")))
    ((string= (company job) "State Police")
      (jzon:write-string "Classified"))
    (t #| Allow default to take over |#
      (call-next-method))))
```

And some examples:

```lisp
(jzon:stringify (make-instance 'job :company "WISE" :title "Agent") :stream t :pretty t)
```
```json
{
  "company": "Eastern Healthcare",
  "title": "Psychologist"
}
```

```lisp
(jzon:stringify (make-instance 'job :company "The Butcher" :title "Assassin") :stream t :pretty t)
```
```json
{
  "company": "City Hall",
  "title": "Clerk",
  "lifelines": [
    "Yuri",
    "Camilla"
  ]
}
```

And something that cannot be done with the other methods:

```lisp
(jzon:stringify (make-instance 'job :company "State Police" :title "Interrogator") :stream t :pretty t)
```
```json
"Classified"
```
## jzon:parser

Similarly to [`jzon:writer`](#jzonwriter), `jzon:parser` exists to parse JSON in parts by providing a simple, SAX-like streaming API.

An example:

```lisp
(jzon:with-parser (parser "{\"x\": 1, \"y\": [2, 3], \"live\": false}")
  (jzon:parse-next parser)  #| :begin-object |#
  (jzon:parse-next parser)  #| :object-key, "x" |#
  (jzon:parse-next parser)  #| :value, 1 |#
  (jzon:parse-next parser)  #| :object-key, "y" |#
  (jzon:parse-next parser)  #| :begin-array |#
  (jzon:parse-next parser)  #| :value, 2 |#
  (jzon:parse-next parser)  #| :value, 3 |#
  (jzon:parse-next parser)  #| :end-array |#
  (jzon:parse-next parser)  #| :object-key, "live" |#
  (jzon:parse-next parser)  #| :value, nil |#
  (jzon:parse-next parser)  #| :end-object |#
  (jzon:parse-next parser)) #| nil |#
```

### jzon:make-parser

*Function* **jzon:make-parser** *in &key max-depth allow-comments allow-trailing-comma max-string-length key-fn*

*=> writer*

* *in* - a string, vector (unsigned-byte 8), stream, or pathname
* *max-depth* - a positive `integer`, or `nil`
* *allow-comments* - a `boolean`
* *allow-trailing-comma* - a `boolean`
* *max-string-length* - a positive `integer`
* *key-fn* - a designator for a function of one argument, or nil

*value* - a `jzon:parser`

#### Description

Creates a parser from `in` for use in subsequent [`jzon:parse-next`](#jzonparse-next).

The behaviour of `jzon:parser` is analogous to `jzon:parse`, except you control the interpretation of the JSON events.

`in` can be any of the following types:

* `string`
* `(vector (unsigned-byte 8))` - octets in utf-8
* `stream` - character or binary in utf-8
* `pathname` - `jzon:make-parder` will open the file for reading in utf-8

:warning: Because [`jzon:make-parser`](#jzonmake-parser) can open a file, it is recommended you use [`jzon:with-parser`](#jzonwith-parser) instead, unless you need indefinite extent.

### jzon:close-parser

*Function* **jzon:close-parser** *parser*

*=> parser*

* *parser* - a [`jzon:parser`](#jzonparser)

#### Description

Closes the parser and releases any held resources.

### jzon:with-parser

*Macro* **jzon:with-parser** *(var &rest args) declaration\* form\**

* *var* - a symbol.
* *declaration* - a declare expression; not evaluated.
* *form* - an implicit progn

#### Description

As [`jzon:make-parser`](#jzonmake-parser) + `unwind-protect` + [`jzon:close-parser`](#jzonclose-parser).

Use this like you would `with-open-file`.

### jzon:parse-next

*Function* **jzon:parse-next** *parser*

*=> event, value*

* *event* - a symbol, see below
* *value* - a `jzon:json-atom`

#### Description

Read the next event from the [`jzon:parser`](#jzonparser).

Always returns two values indicating the next available event on the JSON stream:

| *event*       | *value*                                 |
|---------------|-----------------------------------------|
| :value        | `jzon:json-atom`                        |
| :begin-array  | `nil`                                   |
| :end-array    | `nil`                                   |
| :begin-object | `nil`                                   |
| :object-key   | `simple-string` (depending on *key-fn*) |
| :end-object   | `nil`                                   |
| nil           | `nil`                                   |

**Note:** The `nil` *event* represents conclusion of a toplevel value, and should be taken as "parsing has successfully completed".

### Streaming Parser Example

`jzon:parse` could be approximately defined as follows:

```lisp
(defun my/jzon-parse (in)
  (jzon:with-parser (parser in)
    (let (top stack key)
      (flet ((finish-value (value)
                (typecase stack
                  (null                 (setf top value))
                  ((cons list)          (push value (car stack)))
                  ((cons hash-table)    (setf (gethash (pop key) (car stack)) value)))))
        (loop
          (multiple-value-bind (event value) (jzon:parse-next parser)
            (ecase event
              ((nil)          (return top))
              (:value         (finish-value value))
              (:begin-array   (push (list) stack))
              (:end-array     (finish-value (coerce (the list (nreverse (pop stack))) 'simple-vector)))
              (:begin-object  (push (make-hash-table :test 'equal) stack))
              (:object-key    (push value key))
              (:end-object    (finish-value (pop stack))))))))))
```

# Motivation and Features

In writing jzon, we prioritize the following properties, in order:

* [Safety](#safety)
* [Correctness](#correctness)
* [Simplicity](#simplicity)
* [Performance](#performance)

## Safety

[RFC 8259][JSONRFC] allows setting limits on things such as:

* Number values accepted
* Nesting level of arrays/objects
* Length of strings

We should be safe in the face of untrusted JSON and will error on 'unreasonable' input out-of-the-box, such as deeply nested objects or overly long strings.

### Type Safety

All of jzon's public API's are type safe, issuing `cl:type-error` as appropriate.

Some other JSON parsers will make dangerous use of features like `optimize (safety 0) (speed 3)` without type-checking their public API:

``` lisp
CL-USER> (parse 2)
; Debugger entered on #<SB-SYS:MEMORY-FAULT-ERROR {1003964833}>
```

Such errors are unreasonable.

### Avoid Infinite Interning

jzon chooses to (by default) keep object keys as strings. Some libraries choose to `intern` object keys in some package. This is dangerous in the face of untrusted JSON, as every unique key read will be added to that package and never garbage collected.

### Avoid Stack Exhaustion

`jzon:parse` is written in an iterative way which avoids exhausting the call stack. In addition, we provide `:max-depth` to guard against unreasonable inputs.
For even more control, you can make use of the `jzon:with-parser` API's to avoid consing large amounts of user-supplied data to begin with.

## Correctness

This parser is written against [RFC 8259][JSONRFC] and strives to adhere strictly for maximum compliance and few surprises.

It also has been tested against the [JSONTestSuite][JSONTestSuite]. See the [JSONTestSuite](JSONTestSuite/) directory in this repo for making & running the tests.

In short, jzon is the only CL JSON library which correctly:
* *declines* all invalid inputs per that suite
* *accepts* all valid inputs per that suite

Additionally, jzon is one of a couple which never hard crash due to edge-cases like deeply nested objects/arrays.

### Unambiguous values

Values are never ambiguous between `[]`, `false`, `{}`, `null`, or a missing key.

### Compatible Float IO

While more work is doubtlessly necessary to validate further, care has been taken to ensure floating-point values are not lost between `(jzon:parse (jzon:stringify f))`, even across CL implementations.

In particular, certain edge-case values such as subnormals shall parse `===` with JavaScript parsing libraries.

## Simplicity

You call `jzon:parse`, and you get a reasonably standard CL object back.
You call `jzon:stringify` with a reasonably standard CL object and you should get reasonable JSON.

* No custom data structures or accessors required
* No worrying about key case auto conversion on strings, nor or hyphens/underscores replacement on symbols.
* No worrying about what package symbols are interned in (no symbols).
* No worrying about dynamic variables affecting a parse as in cl-json, jonathan, jsown. Everything affecting `jzon:parse` is given at the call-site.

`jzon:parse` also accepts either a string, octet vector, stream, or pathname for simpler usage over libraries requiring one or the other, or having separate parse functions.

Finally, all public API's strive to have reasonable defaults so things 'Just Work'.

## Performance

While being the last emphasized feature, it is still important to for jzon to perform best-in-class when it comes to reducing parsing times and memory usage.

The general goal benchmark is for jzon to live in the 50% [jsown][jsown] range. This means that if [jsown][jsown] takes 1 second to parse, we will have succeeded if jzon takes <=2 seconds.

With this, jzon will generally outperform all other libraries. 

Importantly, we also strive to be safe even in `(optimize (speed 3) (safety 0))` environments.

### vs jsown

[jsown][jsown] is used as the golden standard when it comes to *performance*, as it offers consistently fast parsing speeds on a wide variety of inputs. 

 However, with jzon, we have **much** higher scores when considering jzon's [priorities](#motivation-and-features).

Consider this REPL interaction when considering [safety](#safety) and [correctness](#correctness):

```
CL-USER(97): (jsown:parse ",1,]")

1
CL-USER(98): (jsown:parse ",1,what]")

1
CL-USER(99): (jsown:parse "[,1,what]")
fatal error encountered in SBCL pid 1238017716:
should not get access violation in dynamic space

Welcome to LDB, a low-level debugger for the Lisp runtime environment.
ldb>
```

[jsown][jsown] will gladly accept blatantly wrong JSON and produce incorrect results. It has faults such as thinking `this` is `true`, and believes `no` is acceptably `null`.

If Performance is you #1 concern, and you're not afraid of your entire process crashing on a stray comma, [jsown][jsown] might be for you.

Please see the JSONTestSuite results with jsown [here](JSONTestSuite/results/jsown.log) for several other failures.

### vs jonathan

[jonathan][jonathan] boasts incredible performance ..  on JSON smaller than 200 bytes, its performance on SBCL tanks on anything larger.

On my machine, parsing a [25MB](https://raw.githubusercontent.com/json-iterator/test-data/master/large-file.json) JSON file, already pre-loaded into a `simple-string`, took over 19 minutes.

In addition, it shares similar correctness issues as jsown, though (usually) not landing me in the ldb:

```
CL-USER(39): (jonathan:parse "12,]???")
CL-USER(40): (jonathan:parse 2)

debugger invoked on a SB-SYS:MEMORY-FAULT-ERROR in thread
#<THREAD "main thread" RUNNING {10010E0073}>:
  Unhandled memory fault at #xFFFFFFFFFFFFFFFD.
```

Please see the JSONTestSuite results with Jonathan [here](JSONTestSuite/results/jonathan.log) for several other failures.

### Object key pooling

By default, jzon will keep track of object keys each `jzon:parse` (or `jzon:make-parser`), causing `string=` keys in a nested JSON object to be shared (`eq`):

```lisp
(jzon:parse "[{\"x\": 5}, {\"x\": 10}, {\"x\": 15}]")
```

In this example, the string `x` is shared (eq) between all 3 objects.

This optimizes for the common case of reading a JSON payload containing many duplicate keys.

**Tip**: This behaviour may be altered by supplying a different `:key-fn` to `jzon:parse` or `jzon:make-parser`.

### `base-string` coercion

When possible, strings will be coerced to `cl:simple-base-string`. This can lead to upwards of 1/4 memory usage per string on implementations like SBCL, which store `string`s internally as UTF32, while `base-string` can be represented in 8 bits per char.

# Dependencies

* [closer-mop](https://github.com/pcostanza/closer-mop)
* [flexi-streams](https://github.com/edicl/flexi-streams)
* [float-features](https://github.com/Shinmera/float-features)
* [uiop](https://gitlab.common-lisp.net/asdf/asdf)

# License

See [LICENSE](LICENSE).

jzon was originally a fork of [st-json](https://marijnhaverbeke.nl/st-json/), but I ended up scrapping all of the code except for for the function decoding Unicode.

# Alternatives

There are many CL JSON libraries available, and I defer to Sabra Crolleton's definitive list and comparisons [https://sabracrolleton.github.io/json-review](https://sabracrolleton.github.io/json-review).

But for posterity, included in this repository is a set of tests and results for the following libraries:

* [cl-json][cl-json]
* [jonathan][jonathan]
* [json-streams][json-streams]
* [jsown][jsown]
* [shasht][shasht]
* [yason][yason]

I believe jzon to be the superior choice and hope for it to become the new, true de-facto library in the world of JSON-in-CL once and for all.

[JSONRFC]: https://tools.ietf.org/html/rfc8259
[JSONTestSuite]: https://github.com/nst/JSONTestSuite
[jsown]: https://github.com/madnificent/jsown
[cl-json]: https://cl-json.common-lisp.dev/cl-json.html
[jonathan]: https://github.com/Rudolph-Miller/jonathan
[json-streams]: https://github.com/rotatef/json-streams
[shasht]: https://github.com/yitzchak/shasht
[yason]: https://github.com/phmarek/yason
