---
layout: publish
title: Int in Python
date: 2024-11-22 22:34:41
tags: [Python]
categories: [Technical]
---

{% asset_img INT.png int %}

About `int` in Python...

<!-- more -->

Ever wonder how Python store integer in memory ?

For me personally, I've never thought about that even though that is the most basic data type being used on a daily basis.

So... yeah, I have no idea how integer is stored in Python 🥲.
After doing some search online, I wanna share what I know till this point.

---

## TL;DR
Python store integer in a bignum fashion separating the signed part and magnitude part.

---

## The low level C object (Python 3.10)
- Integer is an instance of type `PyLongObject` ([ref](https://github.com/python/cpython/blob/3.10/Include/longobject.h))

- `PyLongObject` is a typedef of struct `_longobject`([ref](https://github.com/python/cpython/blob/3.10/Include/longintrepr.h))
    ```c
    typedef struct _longobject PyLongObject;

    /* ---- */

    struct _longobject {
        PyObject_VAR_HEAD
        digit ob_digit[1];
    };
    ```

    Let's take a look at the members of `_longobject`.

1. `digit ob_digit[1];`

    This basically states that an array of digit is used to store the magnitude part of an integer.

    The memory space of type `digit` occupy depends on different system platform (32-bit or 64-bit).

    ```c
    /* 64-bit for example */
    typedef uint32_t digit;
    ```

    Each digit is declared as an unsigned 32-bits integer where is could store values in range of `0 ~ 2^30-1`.

    And the array of digits will be used to calculated the value of an integer using base `2^30`.

    For example, say having this array :
    ```python
    ob_digit = [10, 3, 5]

    # The value of this integer wiil be:
    base = 2**30
    10 * (base)**0 + 3 * (base)**1 + 5 * (base)**2
    ```

    And remember the count of digits of this example is `3`(the length of array is 3) , which will be refered to in the below part.




2. `PyObjectc_VAR_HEAD`

    What consists of `PyObjectc_VAR_HEAD` ? ([ref](https://github.com/python/cpython/blob/3.10/Include/object.h))
    ```c
        #define PyObject_VAR_HEAD      PyVarObject ob_base;

        typedef struct {
            PyObject ob_base;
            Py_ssize_t ob_size; /* Number of items in variable part */
        } PyVarObject;

        #define PyObject_HEAD          PyObject ob_base;

        typedef struct _object {
            _PyObject_HEAD_EXTRA
            Py_ssize_t ob_refcnt;
            PyTypeObject *ob_type;
        } PyObject;
    ```

    Two important information this `PyVarObject` carry on :
    - `ob_base`

        This refers to `PyObject` where it carrys the `reference count` and `object type` information which all objects in Python have.
    - `ob_size`

        The `ob_size` is a signed integer indicating how many digit construct the magnitude part and the sign of an integer.

        So take the example array above :
        ```python
        ob_digit = [10, 3, 5]
        ```

        If this integer is a positive integer , then the `ob_size` will be stored as signed integer `3` , since there are three digits in the array.

    What about the type `Py_ssize_t` ?
    Let's check what document say:
    > A new type Py_ssize_t is introduced,   
    > which has the same size as the compiler’s size_t type, but is signed.   
    > It will be a typedef for ssize_t where available.

    ([ref](https://peps.python.org/pep-0353/))

    also this:
    > A signed integral type such that sizeof(Py_ssize_t) == sizeof(size_t).    
    > C99 doesn’t define such a thing directly
    > (size_t is an unsigned integral type).    
    > See PEP 353 for details.    
    > PY_SSIZE_T_MAX is the largest positive value of type Py_ssize_t.

    ([ref](https://docs.python.org/3.10/c-api/intro.html#c.Py_ssize_t))

    Long story short 🥸:

    Usually in a 64-bit platform, the unsigned int type `size_t` consumes 8-bytes. So the `Py_ssize_t` will also take that much space.

    But since it's an signed integer, the maximum will be `2^63-1`.
    - note

        For 64 bits in 2's compliment expression, one bit is used to indicate sign ,

        so for the rest 63 bits the maximum positive value is `2^63-1`

        [ref](https://docs.python.org/3/library/sys.html#sys.maxsize)

        ```sh
        >>> import sys
        >>> sys.maxsize
        9223372036854775807
        >>> 2**63-1
        9223372036854775807
        ```

### Check the size of integer

If we check how many bytes integer `1` occupy,

it takes 28 bytes (including reference count. object type, ob_size, ob_digit)

```sh
>>> sys.getsizeof(1)
28
```

We then check `2^32-1` and it takes 4 more bytes.

If the maximum value of one digit we mentioned above is `2^30-1` using 4 bytes,

another digit is needed since `2^32-1` is larger than `2^30-1`.

```sh
>>> sys.getsizeof(2**32-1)
32
>>> sys.getsizeof(0xFFFFFFFF)
32
```

---

## About built-in function `bin`

Just a quick thought on this `bin` function.

I was curious about why it shows the binary string for a negative integer (say `-5`) like this `-0b101`.

```sh
>>> bin(-5)
'-0b101'
```

and I personally feel like this expression might has something to do with the sign-magnitude method Python used to stored an integer.


---

That's all I wanna share in this post ~

Maybe some parts mentioned is not 100% correct but just wanna share the process of researching this topic. 🧘

(first time writing post in English, forgive me if you see any error or bug 😅)

See ya ~ 👋
