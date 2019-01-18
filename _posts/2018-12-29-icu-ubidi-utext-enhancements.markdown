---
layout: post
title:  "ICU UBiDi and UText Enhancements"
date:   2018-12-29 12:00:00 -0700
categories: [C++]
tags: [ICU, Text]
---
Currently there are few complete implementations of the Unicode Standard Annex #9: The Bidirectional Algorithm. The most popular include International Components for Unicode (ICU), FriBiDi, Uniscribe, DirectWrite, and Core Text. Uniscribe and DirectWrite and Windows are only, and Core Text is OSX only. Of the remaining two, ICU’s UBiDi implementation (UBiDi) is the most desirable due the to additional library functionality available in a single package. One disadvantage of UBiDi is that is only functions on UTF-16 (UChar) arrays. In other functions within the ICU library, the UText abstraction facility is used to allow any text storage and encoding provider to be used, however, UBiDi currently does not support UText. One of the reasons for the lack of support is the ubidi_writeReordered() and ubidi_writeReverse() functions which write to the provided UChar arrays. The implementation of the UChar UText Provider currently does not support write operations.

Allowing UBiDi to support UText closes an important gap in developing a text stack for displaying and manipulating text in a platform independent way. Currently the use of UBiDi for encodings other than UTF-16 requires conversion to and from a UTF-16 buffer which can increase memory usage. The ability for UBiDi to support UText allows for the use of a single abstracted storage buffer right up to the point where glyph mapping and shaping occur. It also decreases the complexity of having to map index locations for bidirectional between encodings.

The information in this post is relevant to ICU 63.1.

A Ticket has been submitted to correspond to this post: [ICU-20361](https://unicode-org.atlassian.net/browse/ICU-20361 "ICU-20361") UBiDi and UText Enhancements Roll-up

This work should cover the following tickets:
* [ICU-5544](https://unicode-org.atlassian.net/browse/ICU-5544 "ICU-5544") RFE: Add UText support to bidi APIs
* [ICU-6861](https://unicode-org.atlassian.net/browse/ICU-6861 "ICU-6861") UText, handling of errors on open()
* [ICU-11472](https://unicode-org.atlassian.net/browse/ICU-11472 "ICU-11472") Regular Expressions, check 64 bit indexing
* [ICU-12987](https://unicode-org.atlassian.net/browse/ICU-12987 "ICU-12987") UText, better testing of invalid UTF-8
* [ICU-12888](https://unicode-org.atlassian.net/browse/ICU-12888 "ICU-12888") UText failures with malformed UTF-8 input
* [ICU-13481](https://unicode-org.atlassian.net/browse/ICU-13481 "ICU-13481") UText, remove duplicated OPEN state.
* [ICU-20258](https://unicode-org.atlassian.net/browse/ICU-20258 "ICU-20258") Missing information about header and library in API documentation of UText

**Currently this work is being submitted to ICU for Acceptance and Review. Links below point to Repository/Branch of future Pull Request.**

# Design Goals and Principles

The design goals of this enhancement implementation consist of a main goal, and a sub-goal. The main goal is to have UBiDi to support UText, the sub goal is to allow the UChar UText Provider to support write operations. The UText sub-goal is a prerequisite to the main goal and only by satisfying the sub-goal, it will be possible to satisfy the main goal.

In supporting the design goal, additional design principles are considered. These include:
* Minimize the changes, however, be bold enough to make important changes that made sense and were well covered by the test suite.
* Re-use of existing code. For example, the use of macros from utext.h should be employed rather than have duplicate/local code to handle encoding.
* Reduction of existing duplicate code. For example, where there are multiple returns with the same error handling have logic pass through and perform error handling once for all code paths at a common return point.
* Reduce the amount of memory required over speed of execution.
* The UText UChar Provider must be stack allocable without any heap allocations. This will allow it to be used to wrap functions that are passed UChar* parameters safely without need to address memory leaks.

# Implementation

The main goal of having UBiDi support UText is not simply making changes to the UBiDi API to accept UText, but instead to have UBiDi use UText internally for all of its operations. This means that the existing API functions with UChar* parameters become pass-through functions that wrap the parameters in a UText and call a new API function with UText* parameters. 

The following UBiDi functions would require new API functions with UText* parameters, and each of these functions would have their parameters wrapped in a UText and the new API function would be called instead in order to retain backwards compatibility:

```c++
void ubidi_setContext(UBiDi *pBiDi,
                      const UChar *prologue, int32_t proLength,
                      const UChar *epilogue, int32_t epiLength,
                      UErrorCode *pErrorCode);

void ubidi_setPara(UBiDi *pBiDi, const UChar *text, int32_t length,
                   UBiDiLevel paraLevel, UBiDiLevel *embeddingLevels,
                   UErrorCode *pErrorCode);

const UChar * ubidi_getText(const UBiDi *pBiDi);

UBiDiDirection ubidi_getBaseDirection(const UChar *text, int32_t length);

int32_t ubidi_writeReordered(UBiDi *pBiDi,
                             UChar *dest, int32_t destSize,
                             uint16_t options,
                             UErrorCode *pErrorCode);

int32_t ubidi_writeReverse(const UChar *src, int32_t srcLength,
                           UChar *dest, int32_t destSize,
                           uint16_t options,
                           UErrorCode *pErrorCode);
```

It is proposed that the following API functions be added to UBiDi:

```c++
void ubidi_setUContext(UBiDi *pBiDi,
                       UText *prologueUt,
                       UText *epilogueUt,
                       UErrorCode *pErrorCode);

void ubidi_setUPara(UBiDi *pBiDi, UText *pText,
                    UBiDiLevel paraLevel, UBiDiLevel *embeddingLevels,
                    UErrorCode *pErrorCode);

UBiDiDirection ubidi_getUBaseDirection(UText *ut);

const UText * ubidi_getUText(const UBiDi *pBiDi);

int32_t ubidi_writeUReordered(UBiDi *pBiDi,
                              UText *dstUt,
                              uint16_t options,
                              UErrorCode *pErrorCode);

int32_t ubidi_writeUReverse(UText *srcUt,
                            UText *dstUt,
                            uint16_t options,
                            UErrorCode *pErrorCode);
```

The “U” in each API function is used to differentiate it as a UText version separate from the UChar* version as is the general approach in the rest of the library. These functions become the new entry points into UBiDi functionality.

Converting UBiDi over to use UText involves changes to the struct UBiDi in ubidiimp.h. The change involves removing the existing UChar pointer, and replacing it with a UText. A clone is taken of the UText that is passed in so that it can be owned by the struct UBiDi. This clone is shallow so changes to the underlying text would cause the struct UBiDi to become invalid which is inline with the existing approach using UChar*.

In order to retain backwards compatibility a new (different) UChar * pointer is added to struct UBiDi which is returned by ubidi_getText() only if the UChar * version of the API functions were called, otherwise this will return NULL. ubidi_getUText() is the preferred way to retrieve the text associated with a struct UBiDi and should always return a valid UText.

Changes are required to the following files to support the internal use of UText:
* [source/common/unicode/ubidi.h](https://github.com/werbicki/icu/blob/UBiDi-and-UText-Enhancements/icu4c/source/common/unicode/ubidi.h "source/common/unicode/ubidi.h")
* [source/common/ubidiimp.h](https://github.com/werbicki/icu/blob/UBiDi-and-UText-Enhancements/icu4c/source/common/ubidiimp.h "source/common/ubidiimp.h")
* [source/common/ubidi.cpp](https://github.com/werbicki/icu/blob/UBiDi-and-UText-Enhancements/icu4c/source/common/ubidi.cpp "source/common/ubidi.cpp")
* [source/common/ubidiln.cpp](https://github.com/werbicki/icu/blob/UBiDi-and-UText-Enhancements/icu4c/source/common/ubidiln.cpp "source/common/ubidiln.cpp")
* [source/common/ubidiwrt.cpp](https://github.com/werbicki/icu/blob/UBiDi-and-UText-Enhancements/icu4c/source/common/ubidiwrt.cpp "source/common/ubidiwrt.cpp")

In making changes to the ubidi_writeReordered() and ubidi_writeReverse() functions, the deficiencies of the existing UText UChar Provider became apparent. This necessitates work on the existing UText UChar Provider.

UText has been stable since version 3.2 according to the documentation. There are providers for UChar arrays (UTF-16), uint8_t arrays (UTF-8), UnicodeString, CharacterIterators, and Replaceable. Only UnicodeString, and Replaceable currently have write functionality.

The following UText functions would require additional API functions to support an entry point into the write functionality:

```c++
U_STABLE UText * U_EXPORT2
utext_openUChars(UText *ut, 
                 const UChar *s, int64_t length, 
                 UErrorCode *status);

U_STABLE UText * U_EXPORT2
utext_openUTF8(UText *ut, 
               const char *s, int64_t length, 
               UErrorCode *status);
```

The existing functions would have their parameters passed to the new API function in order to retain backwards compatibility:

Looking at the UnicodeString UText Provider, the work “Const” is used to differentiate between read-only and read/write entry points in the UnicodeString. Given the different naming for the UChar and uint8_t UText Providers, this is not possible since both are read-only yet do not use the Const keyword. Therefore, a new set of consistent names is proposed:

```c++
UText * utext_openConstU16(UText *ut, 
                           const UChar *s, int64_t length, int64_t capacity, 
                           UErrorCode *status);

UText * utext_openU16(UText *ut, 
                      UChar *s, int64_t length, int64_t capacity, 
                      UErrorCode *status);

UText * utext_openConstU8(UText *ut, 
                          const uint8_t *s, int64_t length, int64_t capacity, 
                          UErrorCode *status);

UText * utext_openU8(UText *ut, 
                     uint8_t *s, int64_t length, int64_t capacity, 
                     UErrorCode *status);

UText * utext_openConstU32(UText *ut, 
                           const UChar32 *s, int64_t length, int64_t capacity, 
                           UErrorCode *status);

UText * utext_openU32(UText *ut, 
                      UChar32 *s, int64_t length, int64_t capacity, 
                      UErrorCode *status);
```

The use of the work “Const” corresponds to the const array parameter passed into the UText Provider. The use of the “U#” matches the use of “U#” in the utf8.h/utf16.h files where it is shorthand for UTF-8 (U8), and UTF-16 (U16). Additionally a UText UChar32 Provider for UTF-32 (U32) completes all of the array encodings that would be desirable (UTF-32 can be a good storage mechanism for glyph manipulation).

These functions become the new entry points into UText Provider functionality and provide a clean entry into the internally supported Providers.

Where these functions deviate from the current API functions is in the capacity parameter. A lot of thought was put into the addition of this parameter to support an array guard on the total length of the array passed independent of the length. Capacity would work like length in that it can be passed a -1 for the “Const” versions of the new API functions and ignored, or if supplied it will guard the detection of the length of the string. This allows existing API functions to retain backwards compatibility. For the new read/write API functions, of which there are no existing API functions, the capacity parameter must be supplied (0 is a valid value). This will ensure that the Replace/Copy UText functionality cannot grow beyond the declared bounds of the array.

Only a single 64-bit value is provided to UText developers, whereas with a capacity value in 64-bit that is independent of the length, a second 64-bit value is required. In support of the capacity functionality, as well as the ability to fully support 64-bit there are a couple of approaches to the struct UText that can be taken:
* Add a second 64-bit field to struct UText
* Use a union to allow the space currently occupied by the two 32-bit fields to be used as the second 64-bit field
* Treat the first 32-bit field as an address to a 64-bit Integer and allow it to occupy the space of both 32-bit Integers

The first approach would cause the size of struct UText to change, something that would require a lot of thought by people with more experience and history with UText to decide!

The second approach is in fact a no-impact change to the struct UText. It would provide a second 64-bit value, and at the same time maintain no change in the size and layout of struct UText, by emplying a union to overload the UText->b, UText->c values into a single 64-bit value called UText->d:

```c++
/**
  * (protected) 64-bit integer field reserved for use by the text provider.
  * Not used by the UText framework, or by the client (user) of the UText.
  * @stable ICU 3.4
  */
int64_t         a;

union
{
  struct {
    /**
      * (protected) 32-bit integer field reserved for use by the text provider.
      * Not used by the UText framework, or by the client (user) of the UText.
      * @stable ICU 3.4
      */
    int32_t         b;

    /**
      * (protected) 32-bit integer field reserved for use by the text provider.
      * Not used by the UText framework, or by the client (user) of the UText.
      * @stable ICU 3.4
      */
    int32_t         c;
  };

  /**
    * (protected) 64-bit integer field reserved for use by the text provider.
    * Not used by the UText framework, or by the client (user) of the UText.
    * Can be used instead of b, c in the union when extra 64-bit integer is needed.
    * @stable 
    */
  int64_t         d;
};
```

It becomes the responsibility of the UText developer to use either UText->b and UText->c, or UText->d and to know that they overlap in memory. From general inspection it appears that the use of these are exclusive and overlap does not occur.

Although this works for C++, this approach is not ANSI-C compliant, and causes an error when compiling. This is very problemtatic with gendict, and cintltst which uses UText and is ANSI-C compliant when compiling.

Therefore, the third approach of overloading the pointer to UText->b to treat UText->b and UText->c as 64-bit of memory is used.

An addition to the UText library functions is also proposed to provide a validity check for a UText. This is used internally by UText library functions to determine if the UText provided is valid before performing the operation. By exposing this function, library callers have the ability to valid the UText and handle exceptions prior to making any UText library function calls.

```c++
UBool utext_isValid(const UText* ut);
```

Finally, it is proposed that the shallowTextClone() function that is local to the utext.cpp file be promoted to an API function to provide a consistent way for UText Providers to implement thier UTextCLose functions.

```c++
UText *utext_shallowClone(UText * dest, const UText * src, UErrorCode * status);
```

Changes are required to the following files:
* [source/common/unicode/utext.h](https://github.com/werbicki/icu/blob/UBiDi-and-UText-Enhancements/icu4c/source/common/unicode/utext.h "source/common/unicode/utext.h")
* [source/common/utext.cpp](https://github.com/werbicki/icu/blob/UBiDi-and-UText-Enhancements/icu4c/source/common/utext.cpp "source/common/utext.cpp")
* [source/test/cintltst/utxttest.c](https://github.com/werbicki/icu/blob/UBiDi-and-UText-Enhancements/icu4c/source/test/cintltst/utexttst.c "source/test/cintltst/utxttest.c")

The existing implementations of UChar and uint8_t UText Providers were not re-used. Instead, new implementations were coded with remnants of the existing code. Full support for 64-bit lengths is included in each implementation. Separate tracking of length vs. capacity of arrays was also implemented. Extensive comments were used to help future UText Provider Developers in understanding the internal workings of a UText abstraction. A seperate guide to writing a UText Provider is the subject of the next post on this blog.

Use of the chunkContents was added to the UChar UText Provider to allow for 32-bit addressing with 64-bit length/capacity.

The existing uint8_t UText Provider was simplified. Buffers are no longer directionally filled from native indexes outside of the chunkContents but are instead based on predictable chunks and are always filled forward. The Access/Extract UText functionality uses utf8.h macros safety functions and duplicate internal versions of similar code is removed.

The Replace/Copy algorithms were designed to perform in-place string manipulation without requiring any buffers on the stack or heap. Although allocating separate buffers might be slightly quicker, the minimization of memory usage was preferred.

While coding the new U8, U16, and U32 UText Providers, common utext_*() functions were modified to add checks and guards. Each function verifies that the UText passed is valid. Calls to UText->pFuncs functions are checked before they are made. chunkOffset-- manipulations are check for > 0. Functions were reorganized for a single return at the bottom of each call which can allow for easier compiler optimizations.

Although these changes are beyond the minimum required to support using UText in UBiDi, given the age of the UText implementations, the improvements to surrounding code over the years, and the lack of appeal that something as mundane, yet foundational, as UText has, it was determined that the effort should be made now to modernize UText as much as possible. UText may not receive attention otherwise, and it is a very useful and important part, of the ICU library!

The test suite for both UBiDi and UText is a combination of cinltest and intltest. cintltest performs extensive testing of the ubidi_writeReordered() and ubidi_writeReverse() functions, but lacks much meat in the testing of UText. intltest performs the remaining UBiDi conformance testing, and contains a significant test suite for UText.

It was determined that the existing test suite for UBiDi was sufficient, but that given the changes in UText, relying on intltest for testing UText was not enough testing coverage. Significant additional test cases would be required.

# Coding and Testing

All of the changes described in this document are complete, tested, and available in a GIT pull request.

Although the changes sound significant, the only files which were significantly changed from their originals are source/common/utext.cpp, and source/test/cintltst/utxttest.c.

Changes to the testing of UText in cintltst started with simple test cases for Open, Access, Extract, Replace and Copy to exercise parameter validation, algorithms, and returns of basic cases. Next, the test suite for UText from intltest was duplicated, and the dependency on UnicodeString for reference comparisons was removed. Test suites to exercise the UText API, U8 UText Provider, U16 UText Provider, and U32 UText Provider were added increasing the number of test iterations by ~150,000.

# Conclusion

This work addresses areas of UBiDi and UText that have the potential to unlock further improvements. UBiDi confirmance to UText may allow for UBiDi to also be extended to confirm with UBreak and the BreakIterator interface. Also, with full functionality support in UText for writable buffers, other areas of the ICU library may be more easily standardized on using UText.

**This work has been useful to me, and I would like the opportunity to have it reviewed to see if it is fit for inclusion in ICU.**