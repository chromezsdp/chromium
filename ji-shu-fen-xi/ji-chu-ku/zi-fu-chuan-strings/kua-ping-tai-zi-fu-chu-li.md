---
description: chrome跨平台字符处理
---

# 跨平台字符处理

使用计算机的过程中经常会看到一堆莫名其妙的符号（乱码），导致出现乱码的一个重要原因就是未能按照正确的编码格式对字符串解码。当出现两种编码方式不兼容的字符时就会出现乱码，为了正确的显示字符串就必须将其转换为对应的编码。以下介绍了chrome中实现的跨平台编码转换方式。

跨平台字符串处理涉及宽字节表示法与UTF-8表示法之间的转换和宽字节表示法与多字节表示法之间的转换。

## 相关文件

* base/strings/sys\_string\_conversions.h  // 方法定义
* base/strings/sys\_string\_conversions\_win.cc  // windows系统下字符串处理
* base/strings/sys\_string\_conversions\_mac.mm  // mac系统下字符串处理
* base/strings/sys\_string\_conversions\_posix.cc  // 兼容posix系统下字符串处理

## 方法定义

首先定义了宽字节与UTF-8之间的转换，紧接着是宽字节与多字节之间的转换。根据不同系统又分别定义了windows、mac、posix下的转换方法。

使用时不用关系不同系统下的具体实现方式，只需要在前四个方法中选择需要的方法即可。chrome的这种方式极大的降低跨平台开发时字符串编码转换难度。

```cpp
// base/strings/sys_string_conversions.h
namespace base {

// Converts between wide and UTF-8 representations of a string. On error, the
// result is system-dependent.
BASE_EXPORT std::string SysWideToUTF8(const std::wstring& wide)
    WARN_UNUSED_RESULT;
BASE_EXPORT std::wstring SysUTF8ToWide(StringPiece utf8) WARN_UNUSED_RESULT;

// Converts between wide and the system multi-byte representations of a string.
// DANGER: This will lose information and can change (on Windows, this can
// change between reboots).
BASE_EXPORT std::string SysWideToNativeMB(const std::wstring& wide)
    WARN_UNUSED_RESULT;
BASE_EXPORT std::wstring SysNativeMBToWide(StringPiece native_mb)
    WARN_UNUSED_RESULT;

// Windows-specific ------------------------------------------------------------

#if defined(OS_WIN)

// Converts between 8-bit and wide strings, using the given code page. The
// code page identifier is one accepted by the Windows function
// MultiByteToWideChar().
BASE_EXPORT std::wstring SysMultiByteToWide(StringPiece mb, uint32_t code_page)
    WARN_UNUSED_RESULT;
BASE_EXPORT std::string SysWideToMultiByte(const std::wstring& wide,
                                           uint32_t code_page)
    WARN_UNUSED_RESULT;

#endif  // defined(OS_WIN)

// Mac-specific ----------------------------------------------------------------

#if defined(OS_APPLE)

// Converts between STL strings and CFStringRefs/NSStrings.

// Creates a string, and returns it with a refcount of 1. You are responsible
// for releasing it. Returns NULL on failure.
BASE_EXPORT ScopedCFTypeRef<CFStringRef> SysUTF8ToCFStringRef(StringPiece utf8)
    WARN_UNUSED_RESULT;
BASE_EXPORT ScopedCFTypeRef<CFStringRef> SysUTF16ToCFStringRef(
    StringPiece16 utf16) WARN_UNUSED_RESULT;

// Same, but returns an autoreleased NSString.
BASE_EXPORT NSString* SysUTF8ToNSString(StringPiece utf8) WARN_UNUSED_RESULT;
BASE_EXPORT NSString* SysUTF16ToNSString(StringPiece16 utf16)
    WARN_UNUSED_RESULT;

// Converts a CFStringRef to an STL string. Returns an empty string on failure.
BASE_EXPORT std::string SysCFStringRefToUTF8(CFStringRef ref)
    WARN_UNUSED_RESULT;
BASE_EXPORT std::u16string SysCFStringRefToUTF16(CFStringRef ref)
    WARN_UNUSED_RESULT;

// Same, but accepts NSString input. Converts nil NSString* to the appropriate
// string type of length 0.
BASE_EXPORT std::string SysNSStringToUTF8(NSString* ref) WARN_UNUSED_RESULT;
BASE_EXPORT std::u16string SysNSStringToUTF16(NSString* ref) WARN_UNUSED_RESULT;

#endif  // defined(OS_APPLE)

}  // namespace base
```

## 跨平台实现

### windows

windows下使用了两个关键的api分别是，MultiByteToWideChar、WideCharToMultiByte。chrome基于这两个函数实现了宽字节与UTF-8、宽字节与多字节，在windows系统下的转换。

```cpp
// base/strings/sys_string_conversions_win.cc
namespace base {

// Do not assert in this function since it is used by the asssertion code!
std::string SysWideToUTF8(const std::wstring& wide) {
  return SysWideToMultiByte(wide, CP_UTF8);
}

// Do not assert in this function since it is used by the asssertion code!
std::wstring SysUTF8ToWide(StringPiece utf8) {
  return SysMultiByteToWide(utf8, CP_UTF8);
}

std::string SysWideToNativeMB(const std::wstring& wide) {
  return SysWideToMultiByte(wide, CP_ACP);
}

std::wstring SysNativeMBToWide(StringPiece native_mb) {
  return SysMultiByteToWide(native_mb, CP_ACP);
}

// Do not assert in this function since it is used by the asssertion code!
std::wstring SysMultiByteToWide(StringPiece mb, uint32_t code_page) {
  if (mb.empty())
    return std::wstring();

  int mb_length = static_cast<int>(mb.length());
  // Compute the length of the buffer.
  int charcount = MultiByteToWideChar(code_page, 0,
                                      mb.data(), mb_length, NULL, 0);
  if (charcount == 0)
    return std::wstring();

  std::wstring wide;
  wide.resize(charcount);
  MultiByteToWideChar(code_page, 0, mb.data(), mb_length, &wide[0], charcount);

  return wide;
}

// Do not assert in this function since it is used by the asssertion code!
std::string SysWideToMultiByte(const std::wstring& wide, uint32_t code_page) {
  int wide_length = static_cast<int>(wide.length());
  if (wide_length == 0)
    return std::string();

  // Compute the length of the buffer we'll need.
  int charcount = WideCharToMultiByte(code_page, 0, wide.data(), wide_length,
                                      NULL, 0, NULL, NULL);
  if (charcount == 0)
    return std::string();

  std::string mb;
  mb.resize(charcount);
  WideCharToMultiByte(code_page, 0, wide.data(), wide_length,
                      &mb[0], charcount, NULL, NULL);

  return mb;
}

}  // namespace base
```

### mac

相对于windows下的实现，mac的实现看上去略现复杂。主要原因在于mac中的字符串涉及到CFString和NSString，它们可以被C++、Objective-C、Swift使用。

```cpp
// base/strings/sys_string_conversions_mac.mm
namespace base {

namespace {

// Convert the supplied CFString into the specified encoding, and return it as
// an STL string of the template type.  Returns an empty string on failure.
//
// Do not assert in this function since it is used by the asssertion code!
template<typename StringType>
static StringType CFStringToSTLStringWithEncodingT(CFStringRef cfstring,
                                                   CFStringEncoding encoding) {
  CFIndex length = CFStringGetLength(cfstring);
  if (length == 0)
    return StringType();

  CFRange whole_string = CFRangeMake(0, length);
  CFIndex out_size;
  CFIndex converted = CFStringGetBytes(cfstring,
                                       whole_string,
                                       encoding,
                                       0,      // lossByte
                                       false,  // isExternalRepresentation
                                       NULL,   // buffer
                                       0,      // maxBufLen
                                       &out_size);
  if (converted == 0 || out_size == 0)
    return StringType();

  // out_size is the number of UInt8-sized units needed in the destination.
  // A buffer allocated as UInt8 units might not be properly aligned to
  // contain elements of StringType::value_type.  Use a container for the
  // proper value_type, and convert out_size by figuring the number of
  // value_type elements per UInt8.  Leave room for a NUL terminator.
  typename StringType::size_type elements =
      out_size * sizeof(UInt8) / sizeof(typename StringType::value_type) + 1;

  std::vector<typename StringType::value_type> out_buffer(elements);
  converted = CFStringGetBytes(cfstring,
                               whole_string,
                               encoding,
                               0,      // lossByte
                               false,  // isExternalRepresentation
                               reinterpret_cast<UInt8*>(&out_buffer[0]),
                               out_size,
                               NULL);  // usedBufLen
  if (converted == 0)
    return StringType();

  out_buffer[elements - 1] = '\0';
  return StringType(&out_buffer[0], elements - 1);
}

// Given an STL string |in| with an encoding specified by |in_encoding|,
// convert it to |out_encoding| and return it as an STL string of the
// |OutStringType| template type.  Returns an empty string on failure.
//
// Do not assert in this function since it is used by the asssertion code!
template<typename InStringType, typename OutStringType>
static OutStringType STLStringToSTLStringWithEncodingsT(
    const InStringType& in,
    CFStringEncoding in_encoding,
    CFStringEncoding out_encoding) {
  typename InStringType::size_type in_length = in.length();
  if (in_length == 0)
    return OutStringType();

  base::ScopedCFTypeRef<CFStringRef> cfstring(CFStringCreateWithBytesNoCopy(
      NULL,
      reinterpret_cast<const UInt8*>(in.data()),
      in_length * sizeof(typename InStringType::value_type),
      in_encoding,
      false,
      kCFAllocatorNull));
  if (!cfstring)
    return OutStringType();

  return CFStringToSTLStringWithEncodingT<OutStringType>(cfstring,
                                                         out_encoding);
}

// Given a StringPiece |in| with an encoding specified by |in_encoding|, return
// it as a CFStringRef.  Returns NULL on failure.
template <typename CharT>
static ScopedCFTypeRef<CFStringRef> StringPieceToCFStringWithEncodingsT(
    BasicStringPiece<CharT> in,
    CFStringEncoding in_encoding) {
  const auto in_length = in.length();
  if (in_length == 0)
    return ScopedCFTypeRef<CFStringRef>(CFSTR(""), base::scoped_policy::RETAIN);

  return ScopedCFTypeRef<CFStringRef>(CFStringCreateWithBytes(
      kCFAllocatorDefault, reinterpret_cast<const UInt8*>(in.data()),
      in_length * sizeof(CharT), in_encoding, false));
}

// Specify the byte ordering explicitly, otherwise CFString will be confused
// when strings don't carry BOMs, as they typically won't.
static const CFStringEncoding kNarrowStringEncoding = kCFStringEncodingUTF8;
#ifdef __BIG_ENDIAN__
static const CFStringEncoding kMediumStringEncoding = kCFStringEncodingUTF16BE;
static const CFStringEncoding kWideStringEncoding = kCFStringEncodingUTF32BE;
#elif defined(__LITTLE_ENDIAN__)
static const CFStringEncoding kMediumStringEncoding = kCFStringEncodingUTF16LE;
static const CFStringEncoding kWideStringEncoding = kCFStringEncodingUTF32LE;
#endif  // __LITTLE_ENDIAN__

}  // namespace

// Do not assert in this function since it is used by the asssertion code!
std::string SysWideToUTF8(const std::wstring& wide) {
  return STLStringToSTLStringWithEncodingsT<std::wstring, std::string>(
      wide, kWideStringEncoding, kNarrowStringEncoding);
}

// Do not assert in this function since it is used by the asssertion code!
std::wstring SysUTF8ToWide(StringPiece utf8) {
  return STLStringToSTLStringWithEncodingsT<StringPiece, std::wstring>(
      utf8, kNarrowStringEncoding, kWideStringEncoding);
}

std::string SysWideToNativeMB(const std::wstring& wide) {
  return SysWideToUTF8(wide);
}

std::wstring SysNativeMBToWide(StringPiece native_mb) {
  return SysUTF8ToWide(native_mb);
}

ScopedCFTypeRef<CFStringRef> SysUTF8ToCFStringRef(StringPiece utf8) {
  return StringPieceToCFStringWithEncodingsT(utf8, kNarrowStringEncoding);
}

ScopedCFTypeRef<CFStringRef> SysUTF16ToCFStringRef(StringPiece16 utf16) {
  return StringPieceToCFStringWithEncodingsT(utf16, kMediumStringEncoding);
}

NSString* SysUTF8ToNSString(StringPiece utf8) {
  return [mac::CFToNSCast(SysUTF8ToCFStringRef(utf8).release()) autorelease];
}

NSString* SysUTF16ToNSString(StringPiece16 utf16) {
  return [mac::CFToNSCast(SysUTF16ToCFStringRef(utf16).release()) autorelease];
}

std::string SysCFStringRefToUTF8(CFStringRef ref) {
  return CFStringToSTLStringWithEncodingT<std::string>(ref,
                                                       kNarrowStringEncoding);
}

std::u16string SysCFStringRefToUTF16(CFStringRef ref) {
  return CFStringToSTLStringWithEncodingT<std::u16string>(
      ref, kMediumStringEncoding);
}

std::string SysNSStringToUTF8(NSString* nsstring) {
  if (!nsstring)
    return std::string();
  return SysCFStringRefToUTF8(reinterpret_cast<CFStringRef>(nsstring));
}

std::u16string SysNSStringToUTF16(NSString* nsstring) {
  if (!nsstring)
    return std::u16string();
  return SysCFStringRefToUTF16(reinterpret_cast<CFStringRef>(nsstring));
}

}  // namespace base
```

### posix

posix下对部分系统又做了不同处理，部分实现不在一下代码片段中。SysWideToNativeMB和SysNativeMBToWide的实现方式能够更容易看清宽字节与多字节转换的本质。

```cpp
// base/strings/sys_string_conversions_posix.cc
namespace base {

std::string SysWideToUTF8(const std::wstring& wide) {
  // In theory this should be using the system-provided conversion rather
  // than our ICU, but this will do for now.
  return WideToUTF8(wide);
}
std::wstring SysUTF8ToWide(StringPiece utf8) {
  // In theory this should be using the system-provided conversion rather
  // than our ICU, but this will do for now.
  std::wstring out;
  UTF8ToWide(utf8.data(), utf8.size(), &out);
  return out;
}

#if defined(SYSTEM_NATIVE_UTF8) || defined(OS_ANDROID)
// TODO(port): Consider reverting the OS_ANDROID when we have wcrtomb()
// support and a better understanding of what calls these routines.

std::string SysWideToNativeMB(const std::wstring& wide) {
  return WideToUTF8(wide);
}

std::wstring SysNativeMBToWide(StringPiece native_mb) {
  return SysUTF8ToWide(native_mb);
}

#else

std::string SysWideToNativeMB(const std::wstring& wide) {
  mbstate_t ps;

  // Calculate the number of multi-byte characters.  We walk through the string
  // without writing the output, counting the number of multi-byte characters.
  size_t num_out_chars = 0;
  memset(&ps, 0, sizeof(ps));
  for (auto src : wide) {
    // Use a temp buffer since calling wcrtomb with an output of NULL does not
    // calculate the output length.
    char buf[16];
    // Skip NULLs to avoid wcrtomb's special handling of them.
    size_t res = src ? wcrtomb(buf, src, &ps) : 0;
    switch (res) {
      // Handle any errors and return an empty string.
      case static_cast<size_t>(-1):
        return std::string();
      case 0:
        // We hit an embedded null byte, keep going.
        ++num_out_chars;
        break;
      default:
        num_out_chars += res;
        break;
    }
  }

  if (num_out_chars == 0)
    return std::string();

  std::string out;
  out.resize(num_out_chars);

  // We walk the input string again, with |i| tracking the index of the
  // wide input, and |j| tracking the multi-byte output.
  memset(&ps, 0, sizeof(ps));
  for (size_t i = 0, j = 0; i < wide.size(); ++i) {
    const wchar_t src = wide[i];
    // We don't want wcrtomb to do its funkiness for embedded NULLs.
    size_t res = src ? wcrtomb(&out[j], src, &ps) : 0;
    switch (res) {
      // Handle any errors and return an empty string.
      case static_cast<size_t>(-1):
        return std::string();
      case 0:
        // We hit an embedded null byte, keep going.
        ++j;  // Output is already zeroed.
        break;
      default:
        j += res;
        break;
    }
  }

  return out;
}

std::wstring SysNativeMBToWide(StringPiece native_mb) {
  mbstate_t ps;

  // Calculate the number of wide characters.  We walk through the string
  // without writing the output, counting the number of wide characters.
  size_t num_out_chars = 0;
  memset(&ps, 0, sizeof(ps));
  for (size_t i = 0; i < native_mb.size(); ) {
    const char* src = native_mb.data() + i;
    size_t res = mbrtowc(nullptr, src, native_mb.size() - i, &ps);
    switch (res) {
      // Handle any errors and return an empty string.
      case static_cast<size_t>(-2):
      case static_cast<size_t>(-1):
        return std::wstring();
      case 0:
        // We hit an embedded null byte, keep going.
        i += 1;
        FALLTHROUGH;
      default:
        i += res;
        ++num_out_chars;
        break;
    }
  }

  if (num_out_chars == 0)
    return std::wstring();

  std::wstring out;
  out.resize(num_out_chars);

  memset(&ps, 0, sizeof(ps));  // Clear the shift state.
  // We walk the input string again, with |i| tracking the index of the
  // multi-byte input, and |j| tracking the wide output.
  for (size_t i = 0, j = 0; i < native_mb.size(); ++j) {
    const char* src = native_mb.data() + i;
    wchar_t* dst = &out[j];
    size_t res = mbrtowc(dst, src, native_mb.size() - i, &ps);
    switch (res) {
      // Handle any errors and return an empty string.
      case static_cast<size_t>(-2):
      case static_cast<size_t>(-1):
        return std::wstring();
      case 0:
        i += 1;  // Skip null byte.
        break;
      default:
        i += res;
        break;
    }
  }

  return out;
}

#endif  // defined(SYSTEM_NATIVE_UTF8) || defined(OS_ANDROID)

}  // namespace base
```

