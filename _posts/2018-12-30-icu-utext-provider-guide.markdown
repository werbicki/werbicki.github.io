---
layout: post
title:  "ICU Custom UText Providers Guide"
date:   2018-12-30 12:00:00 -0700
categories: [C++]
tags: [ICU, Text]
---
UText is an abstraction facility that allows the underlying native storage mechanism to be separate from its consumption by ICU library facilities. It provides for a mechanism for iteration over code points, extraction of UTF-16 encoded text, and the manipulation of the underlying native storage without accessing it directly.

**The [ICU User Guide section on Text Providers](http://userguide.icu-project.org/strings/utext#TOC-Text-Providers) briefly introduces the requirments for developing a UText Provider. The information is fairly sparse and there is an ICU Ticket [ICU-20258](https://unicode-org.atlassian.net/browse/ICU-20258 "ICU-20258") Missing information about header and library in API documentation of UText, that has requested additional infomration on writing a UText Provider. This guide is meant to be a thorough walk-though of the iternals of UText and what it takes to write a compliant UText Provider.**

A UText is a struct that internally includes the following:
* Members for tracking a UText’s status, memory, and state
* Members for tracking the underlying native storage mechanism
* Members for managing, and iterating over, a chunk of text that has been converted to UTF-16
* Members for additional UText specific memory
* A pointer to a list of UText Provider functions

The UText is manipulated directly by utext_*() library functions and therefore its state can change, even on a read operation. Only where a UText is passed as a const can it be assured that the internal state will not change.

Developing a Custom UText Provider allows any storage mechanism to be used directly by ICU library functions. The underlying storage does not have any requirements itself except that it must be able to be converted to and from UTF-16. Otherwise the underlying storage can be continuous, or non-continuous.

To developer a Custom UText Provider, a developer must write a set of functions to manage Opening, Cloning, Closing, Accessing, Extracting, and optionally Replacing and Copying text using the underlying storage mechanism. By providing a minimum of UText Provider functions, a significant number of functionality is then provided by the utext_*() library functions.

The guide walks through the UText Provider functions, how each is used, and what is required by a Custom UText Provider in order to be compatible with the utext_*() library functions.

# Opening a Custom UText Provider

The first step in developing a Custom UText Provider is to provide a function to open the UText and initialize the struct for calling by utext_*() library functions. The signature of this function is UText Provider dependent. For example, to open a new UText on a std::basic_string<char> a developer might provide the following function:

```c++
UText* open_string(UText* ut, std::string* str, UErrorCode *status);
```

Within this function the first step is to call the utext_setup() helper function to initialize the UText struct and allocate any additional memory that might be required by the Custom UText Provider. By passing the number of bytes to reserve, it is possible to allocate extra memory larger than the size of the UText struct that travels with the UText. This extra memory generally holds buffers that are used as UTF-16 text chunks that have been converted from the underlying storage mechanism to UTF-16 (more on this in the section Accessing Text in a Custom UText Provider).

Once the UText has been setup, the next step is to assign the UText Provider functions. This is what uniquely defines the UText and ties it to a UText Provider. These are the functions that will be called at various times by the utext_*() library functions to manipulate the state of the UText as needed. It is the developer’s job to write the UText functions specific to the Custom UText Provider. The UTextFuncs structure works like a virtual table and the UText Provider developer assigns the corresponding functions, and a pointer to this structure is what is assigned to the pFuncs member in the UText struct:

```c++
static const struct UTextFuncs stringFuncs = 
{
    sizeof(UTextFuncs),
    0, 0, 0,             // Reserved alignment padding
    stringUTextClone,
    stringUTextLength,
    stringUTextAccess,
    stringUTextExtract,
    stringUTextReplace,
    stringUTextCopy
    stringUTextMapOffsetToNative,
    stringUTextMapIndexToUTF16,
    stringUTextClose,
    NULL,                // spare 1
    NULL,                // spare 2
    NULL                 // spare 3
};
```

In the example above, each function is written by the developer of a Custom UText Provider where string is the prefix for the corresponding function from the UTextFuncs functions structure.

UTextFuncs functions include:
| Function	                 | Description |
| UTextAccess	             | Set up the Text Chunk associated with this UText so that it includes a requested index position |
| UTextNativeLength	         | Return the full length of the text |
| UTextClone	             | Clone the UText |
| UTextExtract	             | Extract a range of text into a caller-supplied buffer |
| UTextReplace	             | Replace a range of text with a caller-supplied replacement. May expand or shrink the overall text |
| UTextCopy	                 | Move or copy a range of text to a new position |
| UTextMapOffsetToNative	 | Within the current text chunk, translate a UTF-16 buffer offset to an absolute native index |
| UTextMapNativeIndexToUTF16 | Translate an absolute native index to a UTF-16 buffer offset within the current text |
| UTextClose	             | Provider specific close. Free storage as required |

Not every Custom UText Provider requires all the functions.
Finally, assign to the context in the UText struct the underlying storage mechanism. The context member is a void* pointer to the underlying storage mechanism that was passed in to the open() function, and works like a this pointer. When needed within the Custom UText Provider functions the context is casted back to the underlying storage mechanism’s representation so that it can be manipulated.

# Cloning a Custom UText Provider

It is a fairly common practice to clone a UText to have a copy that can be manipulated without concern for the state of the original UText the clone what made from. There are two important flags that must be considered when cloning: shallow vs. deep, and read-only vs. read-write. A shallow clone will use the same physical native memory in the underlying storage mechanism, whereas a deep clone will need to make a copy of the underlying storage mechanism providing new native memory. Read-only vs. read-write allows the clone to be frozen if it is read-write to ensure that the underlying storage cannot be manipulated.

The second step in developing a Custom UText Provider is to provide a clone function. For example, to clone a UText on a std::basic_string<char> a developer might provide the following function:

```c++
static void stringUTextClone(UText* ut);
```

This is the first function that is part of the UTextFuncs that makes up the Custom UText Provider. Within this function the first step is to call the utext_shallowClone() helper function to clone the UText struct by performing a memory copy of the structure into the destination UText, including all of the extra memory that was allocated. The utext_shallowClone()  function will adjust pointers that point to itself, to point to the destination UText instead, however, it does not adjust pointer to memory outside of the UTtext. Therefore, after this call the destination UText is a copy of the source UText, and all the Custom UText Provider specific memory pointers in the destination UText also point to the same locations as the source UText!

This is fine for a shallow clone, however, for a deep clone it will be necessary for the developer to allocate memory and copy the underlying storage into the newly allocated memory. At this point, the destination UText has allocated memory that is no longer attached to the source UText, and so it is important to set a flag indicating this the destination UText owns the memory. This is important for the UTextCose function:

```c++
destUText->providerProperties |= I32_FLAG(UTEXT_PROVIDER_OWNS_TEXT);
```

# Closing a Custom UText Provider

The second step in developing a Custom UText Provider, and the second UTextFuncs function that is required is close. For example, to close a UText on a std::basic_string<char> a developer might provide the following function:

```c++
static void stringTextClose(UText* ut);
```

Using the utext_setup() and utext_shallowClone() means that a call to utext_close() will do most of the work freeing memory so the developer only needs to free memory that was specifically allocated during a deep clone. Fortunately, this is easy thanks to the UTEXT_PROVIDER_OWNS_TEXT flag, otherwise, close is a non-operative call. It is highly recommended that utext_setup() and utext_shallowClone() be used to ensure that a call to utext_close() correctly frees any extra memory associated with the UText struct.

# Accessing Text in a Custom UText Provider

Iteration over the code points within the underlying storage mechanism is the responsibility of the UText Provider’s length, and access functions:

```c++
static int64_t stringUTextLength(UText *ut);

static UBool stringUTextAccess(UText* ut, int64_t nativeIndex, UBool  forward);
```

The length function simply returns the length of the underlying storage, in the underlying storage mechanism’s native count. For example, on a std::basic_string<char> the length simply returns the number of 8-bit bytes.
It is important to note that at this point the UTextFuncs mostly work on native indexes and lengths to lessen the confusion when converting between the underlying storage and UTF-16. The Custom UText Provider works in native units, and the utext_*() library functions manage the alignment between the two at code point boundaries. The access function, therefore, is straightforward if this consideration is kept in mind.
The access function is responsible for providing a chunk of the underlying storage in UTF-16 encoded format as requested. The request is made by passing in an index in the underlying storage mechanism’s native units. Also passed in is the generally iteration direction as a possible optimization when caching information for future access calls. When the direction is not known, a forward iteration direction is assumed.

The access function is called when an index request is made that is outside of the current chunkContents member. Within the UText struct there are several members that drive the code point iteration of a Custom UText Provider:

| Member              | Description |
| chunkContents       | Pointer to a chunk of text in UTF-16 format. May refer either to original storage of the source of the text, or if conversion was required, to a buffer owned by the UText |
| chunkNativeStart    | Native index of the first character in the text chunk |
| chunkNativeLimit    | Native index of the first character position following the current chunk |
| chunkLength         | Length the text chunk (UTF-16 buffer), in UChars |
| nativeIndexingLimit | The highest chunk offset where native indexing and chunk (UTF-16) indexing correspond |
| chunkOffset         | Current iteration position within the text chunk (UTF-16 buffer) |

For example, when a call is made to utext_next32(), the function checks to see if the current chunkOffset is less than the chunkLength, and if so it will return a code point directly from the chunkContents. If the chunkOffset is outside of the chunkContents, the access function is called with a forward iteration request to access chunkNativeLimit.

When the access function is called the nativeIndex needs to be validated to ensure that it is within the underlying storage. Next the nativeIndex needs to be mapped to the underlying storage mechanism to determine a run of the underlying storage that will be converted to UTF-16. The location, and size, of the chunk that will be converted to UTF-16 is complete up to the developer. Generally, this is a small chunk to limit wasted conversions for string with small access. For example, the internal UTF-8 UText Provider uses two buffers each of 40 bytes.

Once a run of the underlying storage is selected, the next step is to convert that to UTF-16. This is where the pExtra void* pointer plays a role. When utext_setup() was called, if extra space was requested, pExtra will be a pointer to memory equal to the extraSize member in bytes. This could represent a struct that contains additional Custom UText Provider specific member variables or could be used as a raw array of UChars. Either way, this is generally the location of the converted underlying storage in UTF-16, and the chunkContents is set to point to this memory.

Text from the underlying storage mechanism is converted to UTF-16, the chunkNativeStart and chunknativeLimit are set to represent the bounds of the run from the underlying storage, the chunkLength in UTF-16 is set, and the chunkOffset is mapped from the nativeIndex into the chunkContents in UTF-16.

It is required that the chunkContents contain complete UTF-16 code points. This means that the chunkContents cannot begin or end in the middle of two 16-bit values that represent one code point. If the chunkContents begin in the middle it is recommend that the chunkContents be moved back to the start of the code point, likewise if it ends in the middle it is recommend that the chunkContents be extended to include the complete code point.

If the mapping between the underlying storage and UTF-16 is one-to-one, this is all that is needed. If, however, the mapping does not align, further work needs to be done. When converting between UTF-8 and UTF-16, one code point can be represented by up to four 8-bit values in UTF-8, and up to two 16-bit values in UTF-16. Therefore, indexing between the chunkContents and the underlying storage may not line up. This is where the nativeIndexingLimit is used.

So long as the indexes between the underlying storage and the chunkContents in UTF-16 line up it is safe to treat them the same, however, when they don’t line up more advanced mapping is needed. The natvieIndexingLimit is the UTF-16 index in the chunkContents that signals that the index mapping is no longer one-to-one. If the mapping is always one-to-one, then the nativeIndexingLimit is equal to the chunkLength and advanced mapping is never used. This is only possible, however, if the conversion to UTF-16 is one-to-one. When using a std::basic_string<char> as the underlying storage, if we assume that the basic_string can only hold ASCII characters it would be possible to make this case (although there are other issues with making that assumption). If the basic_string holds UTF-8 encoded characters, then a misalignment should be assumed, and advanced mapping should be supported.

To support advanced mapping, the mapOffsetToNative and mapIndexToUTF16 function are required:

```c++
static int64_t stringUTextMapOffsetToNative(const UText* ut);

static int32_t stringTextMapIndexToUTF16(const UText* ut, int64_t index64);
```

While converting text from the underlying storage to UTF-16, the developer tracks both the native index and the UTF-16 index. When they get out of sync, nativeIndexingLimit is set to the UTF-16 index, and the difference is tracked between the native index and the UTF-16 index for each UTF-16 index in the chunkContents. Again, the extra space in the UText struct, and the pExtra void* pointer can be used to store an array of these differences. Generally, two arrays are used, one between native and UTF-16, and one in reverse between UTF-16 and native.

When a request is made within the chunkContents, but above the nativeIndexingLimit, depending on the direction either the mapOffsetToNative or the mapIndexToUTF16 function is called. These functions use the arrays that hold the differences between the native and UTF-16 indexes and return the mapped index.

With chunkContents pointing to the converted buffer, mapping arrays filled in, chunk member variables set, and the new offset determined, the access function ensures that there are UTF-16 characters available to be consumed by the utext_*() library functions.

# Extracting Text from a Custom UText Provider

Using the access function it is possible to iterate over a section of text and extract a large number of code points into an array. Since the chunkContents tend to be small, this will require many calls to the access function to refresh the chunkContents. Instead, when extracting large amount of text it is better to use a more efficient function. This is what the extract function is used for:

```c++
static int32_t stringUTextExtract(UText* ut, int64_t nativeStart, int64_t nativeLimit,
                                  UChar* dest, int32_t destCapacity, UErrorCode *status);
```

The difference between using access and extract is that extract will return UTF-16, not code points. Iteration on the returned array will require UTF-16 to code point conversion. The extract function also does not change the chunkContents or the member variables used in the access function, meaning it must not change the iteration position of the UText.

It is important that the extract function produce the same results as the access function, and therefore should use the same conversion code to ensure that both functions can reliably be interchanged depending on the performance level required assuming that the extract function will always be more efficient than the access function.

# Manipulating Text in a Custom UText Provider

It is possible to develop a Custom UText Provider with no manipulation of the underlying storage so long as there is no open function provided that places the UText in a writable state. To provide a UText with read-write functionality, the replace, and copy functions are required:

```c++
static int32_t stringUTextReplace(UText* ut, int64_t nativeStart, int64_t nativeLimit,
                                  const UChar* replText, int32_t replTextLength, UErrorCode *pErrorCode);

static void unistrTextCopy(UText* ut, int64_t nativeStart, int64_t nativeLimit,
                           int64_t destIndex, UBool move, UErrorCode *pErrorCode);
```

These functions work directly on the underlying storage to make the requested change. In the case of the replace function the replacement text may need to be converted into the encoding of the underlying storage mechanism. It is the responsibility of these functions to allocate additional space should the strings grow beyond what was provided or return the appropriate error if growth cannot occur and bounds have been breached. At the successfully completion the iteration position is moved to the end of the manipulated text.

# Testing of a Custom UText Provider

Test suites for internal UText Providers are available in the cintltst and intltest projects. The cintltst project contains ANSI-C tests, and the intltest project contains C++ tests. It is recommended that a developer of a Custom UText Provider look at these tests and replicate them for their own needs. This will ensure that proper checking of the Custom UText Provider functions are preformed and will reduce issues when using ICU library functions with a Custom UText Provider.
