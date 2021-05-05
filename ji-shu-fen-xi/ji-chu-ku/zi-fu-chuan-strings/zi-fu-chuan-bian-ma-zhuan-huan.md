---
description: chrome中字符串编码转换实现方式
---

# 字符串编码转换

字符串编码转换涉及宽字节表示法与UTF-8表示法之间的转换、宽字节表示法与UTF-16表示法之间的转换、UTF-8表示法与UTF-16表示法之间的转换、UTF-16表示法于ASCII表示法之间的转换、ASCII表示法宽字节表示法之间的转换。

## 相关文件

* base/strings/utf\_string\_conversions.h  // 字符串编码转换定义
* base/strings/utf\_string\_conversions.cc  // 字符串编码转换实现
* base/strings/utf\_string\_conversions\_fuzzer.cc // 字符串编码转换实现

## 方法定义

```cpp
// base/strings/utf_string_conversions.h 
namespace base {

// These convert between UTF-8, -16, and -32 strings. They are potentially slow,
// so avoid unnecessary conversions. The low-level versions return a boolean
// indicating whether the conversion was 100% valid. In this case, it will still
// do the best it can and put the result in the output buffer. The versions that
// return strings ignore this error and just return the best conversion
// possible.
BASE_EXPORT bool WideToUTF8(const wchar_t* src, size_t src_len,
                            std::string* output);
BASE_EXPORT std::string WideToUTF8(WStringPiece wide) WARN_UNUSED_RESULT;
BASE_EXPORT bool UTF8ToWide(const char* src, size_t src_len,
                            std::wstring* output);
BASE_EXPORT std::wstring UTF8ToWide(StringPiece utf8) WARN_UNUSED_RESULT;

BASE_EXPORT bool WideToUTF16(const wchar_t* src,
                             size_t src_len,
                             std::u16string* output);
BASE_EXPORT std::u16string WideToUTF16(WStringPiece wide) WARN_UNUSED_RESULT;
BASE_EXPORT bool UTF16ToWide(const char16_t* src,
                             size_t src_len,
                             std::wstring* output);
BASE_EXPORT std::wstring UTF16ToWide(StringPiece16 utf16) WARN_UNUSED_RESULT;

BASE_EXPORT bool UTF8ToUTF16(const char* src,
                             size_t src_len,
                             std::u16string* output);
BASE_EXPORT std::u16string UTF8ToUTF16(StringPiece utf8) WARN_UNUSED_RESULT;
BASE_EXPORT bool UTF16ToUTF8(const char16_t* src,
                             size_t src_len,
                             std::string* output);
BASE_EXPORT std::string UTF16ToUTF8(StringPiece16 utf16) WARN_UNUSED_RESULT;

// This converts an ASCII string, typically a hardcoded constant, to a UTF16
// string.
BASE_EXPORT std::u16string ASCIIToUTF16(StringPiece ascii) WARN_UNUSED_RESULT;

// Converts to 7-bit ASCII by truncating. The result must be known to be ASCII
// beforehand.
BASE_EXPORT std::string UTF16ToASCII(StringPiece16 utf16) WARN_UNUSED_RESULT;

#if defined(WCHAR_T_IS_UTF16)
// This converts an ASCII string, typically a hardcoded constant, to a wide
// string.
BASE_EXPORT std::wstring ASCIIToWide(StringPiece ascii) WARN_UNUSED_RESULT;

// Converts to 7-bit ASCII by truncating. The result must be known to be ASCII
// beforehand.
BASE_EXPORT std::string WideToASCII(WStringPiece wide) WARN_UNUSED_RESULT;
#endif  // defined(WCHAR_T_IS_UTF16)

// The conversion functions in this file should not be used to convert string
// literals. Instead, the corresponding prefixes (e.g. u"" for UTF16 or L"" for
// Wide) should be used. Deleting the overloads here catches these cases at
// compile time.
template <size_t N>
std::u16string WideToUTF16(const wchar_t (&str)[N]) {
  static_assert(N == 0, "Error: Use the u\"...\" prefix instead.");
  return std::u16string();
}

// TODO(crbug.com/1189439): Also disallow passing string constants in tests.
#if !defined(UNIT_TEST)
template <size_t N>
std::u16string ASCIIToUTF16(const char (&str)[N]) {
  static_assert(N == 0, "Error: Use the u\"...\" prefix instead.");
  return std::u16string();
}

// Mutable character arrays are usually only populated during runtime. Continue
// to allow this conversion.
template <size_t N>
std::u16string ASCIIToUTF16(char (&str)[N]) {
  return ASCIIToUTF16(StringPiece(str));
}
#endif

}  // namespace base
```

## 方法实现

```cpp
// base/strings/utf_string_conversions.cc
namespace base {

namespace {

constexpr int32_t kErrorCodePoint = 0xFFFD;

// Size coefficient ----------------------------------------------------------
// The maximum number of codeunits in the destination encoding corresponding to
// one codeunit in the source encoding.

template <typename SrcChar, typename DestChar>
struct SizeCoefficient {
  static_assert(sizeof(SrcChar) < sizeof(DestChar),
                "Default case: from a smaller encoding to the bigger one");

  // ASCII symbols are encoded by one codeunit in all encodings.
  static constexpr int value = 1;
};

template <>
struct SizeCoefficient<char16_t, char> {
  // One UTF-16 codeunit corresponds to at most 3 codeunits in UTF-8.
  static constexpr int value = 3;
};

#if defined(WCHAR_T_IS_UTF32)
template <>
struct SizeCoefficient<wchar_t, char> {
  // UTF-8 uses at most 4 codeunits per character.
  static constexpr int value = 4;
};

template <>
struct SizeCoefficient<wchar_t, char16_t> {
  // UTF-16 uses at most 2 codeunits per character.
  static constexpr int value = 2;
};
#endif  // defined(WCHAR_T_IS_UTF32)

template <typename SrcChar, typename DestChar>
constexpr int size_coefficient_v =
    SizeCoefficient<std::decay_t<SrcChar>, std::decay_t<DestChar>>::value;

// UnicodeAppendUnsafe --------------------------------------------------------
// Function overloads that write code_point to the output string. Output string
// has to have enough space for the codepoint.

// Convenience typedef that checks whether the passed in type is integral (i.e.
// bool, char, int or their extended versions) and is of the correct size.
template <typename Char, size_t N>
using EnableIfBitsAre = std::enable_if_t<std::is_integral<Char>::value &&
                                             CHAR_BIT * sizeof(Char) == N,
                                         bool>;

template <typename Char, EnableIfBitsAre<Char, 8> = true>
void UnicodeAppendUnsafe(Char* out, int32_t* size, uint32_t code_point) {
  CBU8_APPEND_UNSAFE(out, *size, code_point);
}

template <typename Char, EnableIfBitsAre<Char, 16> = true>
void UnicodeAppendUnsafe(Char* out, int32_t* size, uint32_t code_point) {
  CBU16_APPEND_UNSAFE(out, *size, code_point);
}

template <typename Char, EnableIfBitsAre<Char, 32> = true>
void UnicodeAppendUnsafe(Char* out, int32_t* size, uint32_t code_point) {
  out[(*size)++] = code_point;
}

// DoUTFConversion ------------------------------------------------------------
// Main driver of UTFConversion specialized for different Src encodings.
// dest has to have enough room for the converted text.

template <typename DestChar>
bool DoUTFConversion(const char* src,
                     int32_t src_len,
                     DestChar* dest,
                     int32_t* dest_len) {
  bool success = true;

  for (int32_t i = 0; i < src_len;) {
    int32_t code_point;
    CBU8_NEXT(src, i, src_len, code_point);

    if (!IsValidCodepoint(code_point)) {
      success = false;
      code_point = kErrorCodePoint;
    }

    UnicodeAppendUnsafe(dest, dest_len, code_point);
  }

  return success;
}

template <typename DestChar>
bool DoUTFConversion(const char16_t* src,
                     int32_t src_len,
                     DestChar* dest,
                     int32_t* dest_len) {
  bool success = true;

  auto ConvertSingleChar = [&success](char16_t in) -> int32_t {
    if (!CBU16_IS_SINGLE(in) || !IsValidCodepoint(in)) {
      success = false;
      return kErrorCodePoint;
    }
    return in;
  };

  int32_t i = 0;

  // Always have another symbol in order to avoid checking boundaries in the
  // middle of the surrogate pair.
  while (i < src_len - 1) {
    int32_t code_point;

    if (CBU16_IS_LEAD(src[i]) && CBU16_IS_TRAIL(src[i + 1])) {
      code_point = CBU16_GET_SUPPLEMENTARY(src[i], src[i + 1]);
      if (!IsValidCodepoint(code_point)) {
        code_point = kErrorCodePoint;
        success = false;
      }
      i += 2;
    } else {
      code_point = ConvertSingleChar(src[i]);
      ++i;
    }

    UnicodeAppendUnsafe(dest, dest_len, code_point);
  }

  if (i < src_len)
    UnicodeAppendUnsafe(dest, dest_len, ConvertSingleChar(src[i]));

  return success;
}

#if defined(WCHAR_T_IS_UTF32)

template <typename DestChar>
bool DoUTFConversion(const wchar_t* src,
                     int32_t src_len,
                     DestChar* dest,
                     int32_t* dest_len) {
  bool success = true;

  for (int32_t i = 0; i < src_len; ++i) {
    int32_t code_point = src[i];

    if (!IsValidCodepoint(code_point)) {
      success = false;
      code_point = kErrorCodePoint;
    }

    UnicodeAppendUnsafe(dest, dest_len, code_point);
  }

  return success;
}

#endif  // defined(WCHAR_T_IS_UTF32)

// UTFConversion --------------------------------------------------------------
// Function template for generating all UTF conversions.

template <typename InputString, typename DestString>
bool UTFConversion(const InputString& src_str, DestString* dest_str) {
  if (IsStringASCII(src_str)) {
    dest_str->assign(src_str.begin(), src_str.end());
    return true;
  }

  dest_str->resize(src_str.length() *
                   size_coefficient_v<typename InputString::value_type,
                                      typename DestString::value_type>);

  // Empty string is ASCII => it OK to call operator[].
  auto* dest = &(*dest_str)[0];

  // ICU requires 32 bit numbers.
  int32_t src_len32 = static_cast<int32_t>(src_str.length());
  int32_t dest_len32 = 0;

  bool res = DoUTFConversion(src_str.data(), src_len32, dest, &dest_len32);

  dest_str->resize(dest_len32);
  dest_str->shrink_to_fit();

  return res;
}

}  // namespace

// UTF16 <-> UTF8 --------------------------------------------------------------

bool UTF8ToUTF16(const char* src, size_t src_len, std::u16string* output) {
  return UTFConversion(StringPiece(src, src_len), output);
}

std::u16string UTF8ToUTF16(StringPiece utf8) {
  std::u16string ret;
  // Ignore the success flag of this call, it will do the best it can for
  // invalid input, which is what we want here.
  UTF8ToUTF16(utf8.data(), utf8.size(), &ret);
  return ret;
}

bool UTF16ToUTF8(const char16_t* src, size_t src_len, std::string* output) {
  return UTFConversion(StringPiece16(src, src_len), output);
}

std::string UTF16ToUTF8(StringPiece16 utf16) {
  std::string ret;
  // Ignore the success flag of this call, it will do the best it can for
  // invalid input, which is what we want here.
  UTF16ToUTF8(utf16.data(), utf16.length(), &ret);
  return ret;
}

// UTF-16 <-> Wide -------------------------------------------------------------

#if defined(WCHAR_T_IS_UTF16)
// When wide == UTF-16 the conversions are a NOP.

bool WideToUTF16(const wchar_t* src, size_t src_len, std::u16string* output) {
  output->assign(src, src + src_len);
  return true;
}

std::u16string WideToUTF16(WStringPiece wide) {
  return std::u16string(wide.begin(), wide.end());
}

bool UTF16ToWide(const char16_t* src, size_t src_len, std::wstring* output) {
  output->assign(src, src + src_len);
  return true;
}

std::wstring UTF16ToWide(StringPiece16 utf16) {
  return std::wstring(utf16.begin(), utf16.end());
}

#elif defined(WCHAR_T_IS_UTF32)

bool WideToUTF16(const wchar_t* src, size_t src_len, std::u16string* output) {
  return UTFConversion(base::WStringPiece(src, src_len), output);
}

std::u16string WideToUTF16(WStringPiece wide) {
  std::u16string ret;
  // Ignore the success flag of this call, it will do the best it can for
  // invalid input, which is what we want here.
  WideToUTF16(wide.data(), wide.length(), &ret);
  return ret;
}

bool UTF16ToWide(const char16_t* src, size_t src_len, std::wstring* output) {
  return UTFConversion(StringPiece16(src, src_len), output);
}

std::wstring UTF16ToWide(StringPiece16 utf16) {
  std::wstring ret;
  // Ignore the success flag of this call, it will do the best it can for
  // invalid input, which is what we want here.
  UTF16ToWide(utf16.data(), utf16.length(), &ret);
  return ret;
}

#endif  // defined(WCHAR_T_IS_UTF32)

// UTF-8 <-> Wide --------------------------------------------------------------

// UTF8ToWide is the same code, regardless of whether wide is 16 or 32 bits

bool UTF8ToWide(const char* src, size_t src_len, std::wstring* output) {
  return UTFConversion(StringPiece(src, src_len), output);
}

std::wstring UTF8ToWide(StringPiece utf8) {
  std::wstring ret;
  // Ignore the success flag of this call, it will do the best it can for
  // invalid input, which is what we want here.
  UTF8ToWide(utf8.data(), utf8.length(), &ret);
  return ret;
}

#if defined(WCHAR_T_IS_UTF16)
// Easy case since we can use the "utf" versions we already wrote above.

bool WideToUTF8(const wchar_t* src, size_t src_len, std::string* output) {
  return UTF16ToUTF8(as_u16cstr(src), src_len, output);
}

std::string WideToUTF8(WStringPiece wide) {
  return UTF16ToUTF8(StringPiece16(as_u16cstr(wide), wide.size()));
}

#elif defined(WCHAR_T_IS_UTF32)

bool WideToUTF8(const wchar_t* src, size_t src_len, std::string* output) {
  return UTFConversion(WStringPiece(src, src_len), output);
}

std::string WideToUTF8(WStringPiece wide) {
  std::string ret;
  // Ignore the success flag of this call, it will do the best it can for
  // invalid input, which is what we want here.
  WideToUTF8(wide.data(), wide.length(), &ret);
  return ret;
}

#endif  // defined(WCHAR_T_IS_UTF32)

std::u16string ASCIIToUTF16(StringPiece ascii) {
  DCHECK(IsStringASCII(ascii)) << ascii;
  return std::u16string(ascii.begin(), ascii.end());
}

std::string UTF16ToASCII(StringPiece16 utf16) {
  DCHECK(IsStringASCII(utf16)) << UTF16ToUTF8(utf16);
  return std::string(utf16.begin(), utf16.end());
}

#if defined(WCHAR_T_IS_UTF16)
std::wstring ASCIIToWide(StringPiece ascii) {
  DCHECK(IsStringASCII(ascii)) << ascii;
  return std::wstring(ascii.begin(), ascii.end());
}

std::string WideToASCII(WStringPiece wide) {
  DCHECK(IsStringASCII(wide)) << wide;
  return std::string(wide.begin(), wide.end());
}
#endif  // defined(WCHAR_T_IS_UTF16)

}  // namespace base
```

### fuzzer

```cpp
// Entry point for LibFuzzer.
extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
  base::StringPiece string_piece_input(reinterpret_cast<const char*>(data),
                                       size);

  ignore_result(base::UTF8ToWide(string_piece_input));
  base::UTF8ToWide(reinterpret_cast<const char*>(data), size,
                   &output_std_wstring);
  ignore_result(base::UTF8ToUTF16(string_piece_input));
  base::UTF8ToUTF16(reinterpret_cast<const char*>(data), size,
                    &output_string16);

  // Test for char16_t.
  if (size % 2 == 0) {
    base::StringPiece16 string_piece_input16(
        reinterpret_cast<const char16_t*>(data), size / 2);
    ignore_result(base::UTF16ToWide(output_string16));
    base::UTF16ToWide(reinterpret_cast<const char16_t*>(data), size / 2,
                      &output_std_wstring);
    ignore_result(base::UTF16ToUTF8(string_piece_input16));
    base::UTF16ToUTF8(reinterpret_cast<const char16_t*>(data), size / 2,
                      &output_std_string);
  }

  // Test for wchar_t.
  size_t wchar_t_size = sizeof(wchar_t);
  if (size % wchar_t_size == 0) {
    ignore_result(base::WideToUTF8(output_std_wstring));
    base::WideToUTF8(reinterpret_cast<const wchar_t*>(data),
                     size / wchar_t_size, &output_std_string);
    ignore_result(base::WideToUTF16(output_std_wstring));
    base::WideToUTF16(reinterpret_cast<const wchar_t*>(data),
                      size / wchar_t_size, &output_string16);
  }

  // Test for ASCII. This condition is needed to avoid hitting instant CHECK
  // failures.
  if (base::IsStringASCII(string_piece_input)) {
    output_string16 = base::ASCIIToUTF16(string_piece_input);
    base::StringPiece16 string_piece_input16(output_string16);
    ignore_result(base::UTF16ToASCII(string_piece_input16));
  }

  return 0;
}
```

