---
description: chrome中字符串类型转换
---

# 字符串类型转换

字符串类型转换主要设计数字于字符转之间的转换和十六进制编码

## 相关文件

* base/strings/string\_number\_conversions.h  // std::string、std::u16string与数字之间的转换定义
* base/strings/string\_number\_conversions.cc  // std::string、std::u16string与数字之间的转换实现
* base/strings/string\_number\_conversions\_fuzzer.cc  // 数字与字符转转换实现
* base/strings/string\_number\_conversions\_internal.h  // 数字与字符转转换实现
* base/strings/string\_number\_conversions\_win.h  // std::wstring与数字之间的转换定义
* base/strings/string\_number\_conversions\_win.cc  // std::wstring与数字之间的转换实现

## 方法定义

### 数字转换字符串

使用函数重载的方式实现使开发不需要记忆大量的函数名，只需关心需要的字符串类型string或u16string。

### 字符串转换数字

字符串转换数字有可能失败，需要判断StringTo\*返回是否为true。

### 十六进制编码

十六进制表示法在编程中非常常见，将十六进制编码字符转换为对应的数据类型，是对“字符串转换数字”的一个扩充。

```cpp
// base/strings/string_number_conversions.h
namespace base {

// Number -> string conversions ------------------------------------------------

// Ignores locale! see warning above.
BASE_EXPORT std::string NumberToString(int value);
BASE_EXPORT std::u16string NumberToString16(int value);
BASE_EXPORT std::string NumberToString(unsigned int value);
BASE_EXPORT std::u16string NumberToString16(unsigned int value);
BASE_EXPORT std::string NumberToString(long value);
BASE_EXPORT std::u16string NumberToString16(long value);
BASE_EXPORT std::string NumberToString(unsigned long value);
BASE_EXPORT std::u16string NumberToString16(unsigned long value);
BASE_EXPORT std::string NumberToString(long long value);
BASE_EXPORT std::u16string NumberToString16(long long value);
BASE_EXPORT std::string NumberToString(unsigned long long value);
BASE_EXPORT std::u16string NumberToString16(unsigned long long value);
BASE_EXPORT std::string NumberToString(double value);
BASE_EXPORT std::u16string NumberToString16(double value);

// String -> number conversions ------------------------------------------------

// Perform a best-effort conversion of the input string to a numeric type,
// setting |*output| to the result of the conversion.  Returns true for
// "perfect" conversions; returns false in the following cases:
//  - Overflow. |*output| will be set to the maximum value supported
//    by the data type.
//  - Underflow. |*output| will be set to the minimum value supported
//    by the data type.
//  - Trailing characters in the string after parsing the number.  |*output|
//    will be set to the value of the number that was parsed.
//  - Leading whitespace in the string before parsing the number. |*output| will
//    be set to the value of the number that was parsed.
//  - No characters parseable as a number at the beginning of the string.
//    |*output| will be set to 0.
//  - Empty string.  |*output| will be set to 0.
// WARNING: Will write to |output| even when returning false.
//          Read the comments above carefully.
BASE_EXPORT bool StringToInt(StringPiece input, int* output);
BASE_EXPORT bool StringToInt(StringPiece16 input, int* output);

BASE_EXPORT bool StringToUint(StringPiece input, unsigned* output);
BASE_EXPORT bool StringToUint(StringPiece16 input, unsigned* output);

BASE_EXPORT bool StringToInt64(StringPiece input, int64_t* output);
BASE_EXPORT bool StringToInt64(StringPiece16 input, int64_t* output);

BASE_EXPORT bool StringToUint64(StringPiece input, uint64_t* output);
BASE_EXPORT bool StringToUint64(StringPiece16 input, uint64_t* output);

BASE_EXPORT bool StringToSizeT(StringPiece input, size_t* output);
BASE_EXPORT bool StringToSizeT(StringPiece16 input, size_t* output);

// For floating-point conversions, only conversions of input strings in decimal
// form are defined to work.  Behavior with strings representing floating-point
// numbers in hexadecimal, and strings representing non-finite values (such as
// NaN and inf) is undefined.  Otherwise, these behave the same as the integral
// variants.  This expects the input string to NOT be specific to the locale.
// If your input is locale specific, use ICU to read the number.
// WARNING: Will write to |output| even when returning false.
//          Read the comments here and above StringToInt() carefully.
BASE_EXPORT bool StringToDouble(StringPiece input, double* output);
BASE_EXPORT bool StringToDouble(StringPiece16 input, double* output);

// Hex encoding ----------------------------------------------------------------

// Returns a hex string representation of a binary buffer. The returned hex
// string will be in upper case. This function does not check if |size| is
// within reasonable limits since it's written with trusted data in mind.  If
// you suspect that the data you want to format might be large, the absolute
// max size for |size| should be is
//   std::numeric_limits<size_t>::max() / 2
BASE_EXPORT std::string HexEncode(const void* bytes, size_t size);
BASE_EXPORT std::string HexEncode(base::span<const uint8_t> bytes);

// Best effort conversion, see StringToInt above for restrictions.
// Will only successful parse hex values that will fit into |output|, i.e.
// -0x80000000 < |input| < 0x7FFFFFFF.
BASE_EXPORT bool HexStringToInt(StringPiece input, int* output);

// Best effort conversion, see StringToInt above for restrictions.
// Will only successful parse hex values that will fit into |output|, i.e.
// 0x00000000 < |input| < 0xFFFFFFFF.
// The string is not required to start with 0x.
BASE_EXPORT bool HexStringToUInt(StringPiece input, uint32_t* output);

// Best effort conversion, see StringToInt above for restrictions.
// Will only successful parse hex values that will fit into |output|, i.e.
// -0x8000000000000000 < |input| < 0x7FFFFFFFFFFFFFFF.
BASE_EXPORT bool HexStringToInt64(StringPiece input, int64_t* output);

// Best effort conversion, see StringToInt above for restrictions.
// Will only successful parse hex values that will fit into |output|, i.e.
// 0x0000000000000000 < |input| < 0xFFFFFFFFFFFFFFFF.
// The string is not required to start with 0x.
BASE_EXPORT bool HexStringToUInt64(StringPiece input, uint64_t* output);

// Similar to the previous functions, except that output is a vector of bytes.
// |*output| will contain as many bytes as were successfully parsed prior to the
// error.  There is no overflow, but input.size() must be evenly divisible by 2.
// Leading 0x or +/- are not allowed.
BASE_EXPORT bool HexStringToBytes(StringPiece input,
                                  std::vector<uint8_t>* output);

// Same as HexStringToBytes, but for an std::string.
BASE_EXPORT bool HexStringToString(StringPiece input, std::string* output);

// Decodes the hex string |input| into a presized |output|. The output buffer
// must be sized exactly to |input.size() / 2| or decoding will fail and no
// bytes will be written to |output|. Decoding an empty input is also
// considered a failure. When decoding fails due to encountering invalid input
// characters, |output| will have been filled with the decoded bytes up until
// the failure.
BASE_EXPORT bool HexStringToSpan(StringPiece input, base::span<uint8_t> output);

}  // namespace base
```

### windows

```cpp
// base/strings/string_number_conversions_win.h
namespace base {

BASE_EXPORT std::wstring NumberToWString(int value);
BASE_EXPORT std::wstring NumberToWString(unsigned int value);
BASE_EXPORT std::wstring NumberToWString(long value);
BASE_EXPORT std::wstring NumberToWString(unsigned long value);
BASE_EXPORT std::wstring NumberToWString(long long value);
BASE_EXPORT std::wstring NumberToWString(unsigned long long value);
BASE_EXPORT std::wstring NumberToWString(double value);

// The following section contains overloads of the cross-platform APIs for
// std::wstring and base::WStringPiece.
BASE_EXPORT bool StringToInt(WStringPiece input, int* output);
BASE_EXPORT bool StringToUint(WStringPiece input, unsigned* output);
BASE_EXPORT bool StringToInt64(WStringPiece input, int64_t* output);
BASE_EXPORT bool StringToUint64(WStringPiece input, uint64_t* output);
BASE_EXPORT bool StringToSizeT(WStringPiece input, size_t* output);
BASE_EXPORT bool StringToDouble(WStringPiece input, double* output);

}  // namespace base
```

## 方法实现

```cpp
// base/strings/string_number_conversions.cc
namespace base {

std::string NumberToString(int value) {
  return internal::IntToStringT<std::string>(value);
}

std::u16string NumberToString16(int value) {
  return internal::IntToStringT<std::u16string>(value);
}

std::string NumberToString(unsigned value) {
  return internal::IntToStringT<std::string>(value);
}

std::u16string NumberToString16(unsigned value) {
  return internal::IntToStringT<std::u16string>(value);
}

std::string NumberToString(long value) {
  return internal::IntToStringT<std::string>(value);
}

std::u16string NumberToString16(long value) {
  return internal::IntToStringT<std::u16string>(value);
}

std::string NumberToString(unsigned long value) {
  return internal::IntToStringT<std::string>(value);
}

std::u16string NumberToString16(unsigned long value) {
  return internal::IntToStringT<std::u16string>(value);
}

std::string NumberToString(long long value) {
  return internal::IntToStringT<std::string>(value);
}

std::u16string NumberToString16(long long value) {
  return internal::IntToStringT<std::u16string>(value);
}

std::string NumberToString(unsigned long long value) {
  return internal::IntToStringT<std::string>(value);
}

std::u16string NumberToString16(unsigned long long value) {
  return internal::IntToStringT<std::u16string>(value);
}

std::string NumberToString(double value) {
  return internal::DoubleToStringT<std::string>(value);
}

std::u16string NumberToString16(double value) {
  return internal::DoubleToStringT<std::u16string>(value);
}

bool StringToInt(StringPiece input, int* output) {
  return internal::StringToIntImpl(input, *output);
}

bool StringToInt(StringPiece16 input, int* output) {
  return internal::StringToIntImpl(input, *output);
}

bool StringToUint(StringPiece input, unsigned* output) {
  return internal::StringToIntImpl(input, *output);
}

bool StringToUint(StringPiece16 input, unsigned* output) {
  return internal::StringToIntImpl(input, *output);
}

bool StringToInt64(StringPiece input, int64_t* output) {
  return internal::StringToIntImpl(input, *output);
}

bool StringToInt64(StringPiece16 input, int64_t* output) {
  return internal::StringToIntImpl(input, *output);
}

bool StringToUint64(StringPiece input, uint64_t* output) {
  return internal::StringToIntImpl(input, *output);
}

bool StringToUint64(StringPiece16 input, uint64_t* output) {
  return internal::StringToIntImpl(input, *output);
}

bool StringToSizeT(StringPiece input, size_t* output) {
  return internal::StringToIntImpl(input, *output);
}

bool StringToSizeT(StringPiece16 input, size_t* output) {
  return internal::StringToIntImpl(input, *output);
}

bool StringToDouble(StringPiece input, double* output) {
  return internal::StringToDoubleImpl(input, input.data(), *output);
}

bool StringToDouble(StringPiece16 input, double* output) {
  return internal::StringToDoubleImpl(
      input, reinterpret_cast<const uint16_t*>(input.data()), *output);
}

std::string HexEncode(const void* bytes, size_t size) {
  static const char kHexChars[] = "0123456789ABCDEF";

  // Each input byte creates two output hex characters.
  std::string ret(size * 2, '\0');

  for (size_t i = 0; i < size; ++i) {
    char b = reinterpret_cast<const char*>(bytes)[i];
    ret[(i * 2)] = kHexChars[(b >> 4) & 0xf];
    ret[(i * 2) + 1] = kHexChars[b & 0xf];
  }
  return ret;
}

std::string HexEncode(base::span<const uint8_t> bytes) {
  return HexEncode(bytes.data(), bytes.size());
}

bool HexStringToInt(StringPiece input, int* output) {
  return internal::HexStringToIntImpl(input, *output);
}

bool HexStringToUInt(StringPiece input, uint32_t* output) {
  return internal::HexStringToIntImpl(input, *output);
}

bool HexStringToInt64(StringPiece input, int64_t* output) {
  return internal::HexStringToIntImpl(input, *output);
}

bool HexStringToUInt64(StringPiece input, uint64_t* output) {
  return internal::HexStringToIntImpl(input, *output);
}

bool HexStringToBytes(StringPiece input, std::vector<uint8_t>* output) {
  DCHECK(output->empty());
  return internal::HexStringToByteContainer(input, std::back_inserter(*output));
}

bool HexStringToString(StringPiece input, std::string* output) {
  DCHECK(output->empty());
  return internal::HexStringToByteContainer(input, std::back_inserter(*output));
}

bool HexStringToSpan(StringPiece input, base::span<uint8_t> output) {
  if (input.size() / 2 != output.size())
    return false;

  return internal::HexStringToByteContainer(input, output.begin());
}

}  // namespace base
```

### internal

```cpp
// base/strings/string_number_conversions_internal.h
namespace base {

namespace internal {

template <typename STR, typename INT>
static STR IntToStringT(INT value) {
  // log10(2) ~= 0.3 bytes needed per bit or per byte log10(2**8) ~= 2.4.
  // So round up to allocate 3 output characters per byte, plus 1 for '-'.
  const size_t kOutputBufSize =
      3 * sizeof(INT) + std::numeric_limits<INT>::is_signed;

  // Create the string in a temporary buffer, write it back to front, and
  // then return the substr of what we ended up using.
  using CHR = typename STR::value_type;
  CHR outbuf[kOutputBufSize];

  // The ValueOrDie call below can never fail, because UnsignedAbs is valid
  // for all valid inputs.
  std::make_unsigned_t<INT> res =
      CheckedNumeric<INT>(value).UnsignedAbs().ValueOrDie();

  CHR* end = outbuf + kOutputBufSize;
  CHR* i = end;
  do {
    --i;
    DCHECK(i != outbuf);
    *i = static_cast<CHR>((res % 10) + '0');
    res /= 10;
  } while (res != 0);
  if (IsValueNegative(value)) {
    --i;
    DCHECK(i != outbuf);
    *i = static_cast<CHR>('-');
  }
  return STR(i, end);
}

// Utility to convert a character to a digit in a given base
template <int BASE, typename CHAR>
Optional<uint8_t> CharToDigit(CHAR c) {
  static_assert(1 <= BASE && BASE <= 36, "BASE needs to be in [1, 36]");
  if (c >= '0' && c < '0' + std::min(BASE, 10))
    return c - '0';

  if (c >= 'a' && c < 'a' + BASE - 10)
    return c - 'a' + 10;

  if (c >= 'A' && c < 'A' + BASE - 10)
    return c - 'A' + 10;

  return base::nullopt;
}

// There is an IsUnicodeWhitespace for wchars defined in string_util.h, but it
// is locale independent, whereas the functions we are replacing were
// locale-dependent. TBD what is desired, but for the moment let's not
// introduce a change in behaviour.
template <typename CHAR>
class WhitespaceHelper {};

template <>
class WhitespaceHelper<char> {
 public:
  static bool Invoke(char c) {
    return 0 != isspace(static_cast<unsigned char>(c));
  }
};

template <>
class WhitespaceHelper<char16_t> {
 public:
  static bool Invoke(char16_t c) { return 0 != iswspace(c); }
};

template <typename CHAR>
bool LocalIsWhitespace(CHAR c) {
  return WhitespaceHelper<CHAR>::Invoke(c);
}

template <typename Number, int kBase>
class StringToNumberParser {
 public:
  struct Result {
    Number value = 0;
    bool valid = false;
  };

  static constexpr Number kMin = std::numeric_limits<Number>::min();
  static constexpr Number kMax = std::numeric_limits<Number>::max();

  // Sign provides:
  //  - a static function, CheckBounds, that determines whether the next digit
  //    causes an overflow/underflow
  //  - a static function, Increment, that appends the next digit appropriately
  //    according to the sign of the number being parsed.
  template <typename Sign>
  class Base {
   public:
    template <typename Iter>
    static Result Invoke(Iter begin, Iter end) {
      Number value = 0;

      if (begin == end) {
        return {value, false};
      }

      // Note: no performance difference was found when using template
      // specialization to remove this check in bases other than 16
      if (kBase == 16 && end - begin > 2 && *begin == '0' &&
          (*(begin + 1) == 'x' || *(begin + 1) == 'X')) {
        begin += 2;
      }

      for (Iter current = begin; current != end; ++current) {
        Optional<uint8_t> new_digit = CharToDigit<kBase>(*current);

        if (!new_digit) {
          return {value, false};
        }

        if (current != begin) {
          Result result = Sign::CheckBounds(value, *new_digit);
          if (!result.valid)
            return result;

          value *= kBase;
        }

        value = Sign::Increment(value, *new_digit);
      }
      return {value, true};
    }
  };

  class Positive : public Base<Positive> {
   public:
    static Result CheckBounds(Number value, uint8_t new_digit) {
      if (value > static_cast<Number>(kMax / kBase) ||
          (value == static_cast<Number>(kMax / kBase) &&
           new_digit > kMax % kBase)) {
        return {kMax, false};
      }
      return {value, true};
    }
    static Number Increment(Number lhs, uint8_t rhs) { return lhs + rhs; }
  };

  class Negative : public Base<Negative> {
   public:
    static Result CheckBounds(Number value, uint8_t new_digit) {
      if (value < kMin / kBase ||
          (value == kMin / kBase && new_digit > 0 - kMin % kBase)) {
        return {kMin, false};
      }
      return {value, true};
    }
    static Number Increment(Number lhs, uint8_t rhs) { return lhs - rhs; }
  };
};

template <typename Number, int kBase, typename CharT>
auto StringToNumber(BasicStringPiece<CharT> input) {
  using Parser = StringToNumberParser<Number, kBase>;
  using Result = typename Parser::Result;

  bool has_leading_whitespace = false;
  auto begin = input.begin();
  auto end = input.end();

  while (begin != end && LocalIsWhitespace(*begin)) {
    has_leading_whitespace = true;
    ++begin;
  }

  if (begin != end && *begin == '-') {
    if (!std::numeric_limits<Number>::is_signed) {
      return Result{0, false};
    }

    Result result = Parser::Negative::Invoke(begin + 1, end);
    result.valid &= !has_leading_whitespace;
    return result;
  }

  if (begin != end && *begin == '+') {
    ++begin;
  }

  Result result = Parser::Positive::Invoke(begin, end);
  result.valid &= !has_leading_whitespace;
  return result;
}

template <typename CharT, typename VALUE>
bool StringToIntImpl(BasicStringPiece<CharT> input, VALUE& output) {
  auto result = StringToNumber<VALUE, 10>(input);
  output = result.value;
  return result.valid;
}

template <typename CharT, typename VALUE>
bool HexStringToIntImpl(BasicStringPiece<CharT> input, VALUE& output) {
  auto result = StringToNumber<VALUE, 16>(input);
  output = result.value;
  return result.valid;
}

static const double_conversion::DoubleToStringConverter*
GetDoubleToStringConverter() {
  static NoDestructor<double_conversion::DoubleToStringConverter> converter(
      double_conversion::DoubleToStringConverter::EMIT_POSITIVE_EXPONENT_SIGN,
      nullptr, nullptr, 'e', -6, 12, 0, 0);
  return converter.get();
}

// Converts a given (data, size) pair to a desired string type. For
// performance reasons, this dispatches to a different constructor if the
// passed-in data matches the string's value_type.
template <typename StringT>
StringT ToString(const typename StringT::value_type* data, size_t size) {
  return StringT(data, size);
}

template <typename StringT, typename CharT>
StringT ToString(const CharT* data, size_t size) {
  return StringT(data, data + size);
}

template <typename StringT>
StringT DoubleToStringT(double value) {
  char buffer[32];
  double_conversion::StringBuilder builder(buffer, sizeof(buffer));
  GetDoubleToStringConverter()->ToShortest(value, &builder);
  return ToString<StringT>(buffer, builder.position());
}

template <typename STRING, typename CHAR>
bool StringToDoubleImpl(STRING input, const CHAR* data, double& output) {
  static NoDestructor<double_conversion::StringToDoubleConverter> converter(
      double_conversion::StringToDoubleConverter::ALLOW_LEADING_SPACES |
          double_conversion::StringToDoubleConverter::ALLOW_TRAILING_JUNK,
      0.0, 0, nullptr, nullptr);

  int processed_characters_count;
  output = converter->StringToDouble(data, input.size(),
                                     &processed_characters_count);

  // Cases to return false:
  //  - If the input string is empty, there was nothing to parse.
  //  - If the value saturated to HUGE_VAL.
  //  - If the entire string was not processed, there are either characters
  //    remaining in the string after a parsed number, or the string does not
  //    begin with a parseable number.
  //  - If the first character is a space, there was leading whitespace
  return !input.empty() && output != HUGE_VAL && output != -HUGE_VAL &&
         static_cast<size_t>(processed_characters_count) == input.size() &&
         !IsUnicodeWhitespace(input[0]);
}

template <typename OutIter>
static bool HexStringToByteContainer(StringPiece input, OutIter output) {
  size_t count = input.size();
  if (count == 0 || (count % 2) != 0)
    return false;
  for (uintptr_t i = 0; i < count / 2; ++i) {
    // most significant 4 bits
    Optional<uint8_t> msb = CharToDigit<16>(input[i * 2]);
    // least significant 4 bits
    Optional<uint8_t> lsb = CharToDigit<16>(input[i * 2 + 1]);
    if (!msb || !lsb) {
      return false;
    }
    *(output++) = (*msb << 4) | *lsb;
  }
  return true;
}

}  // namespace internal

}  // namespace base
```

### fuzzer

```cpp
// base/strings/string_number_conversions_fuzzer.cc
template <class NumberType, class StringPieceType, class StringType>
void CheckRoundtripsT(const uint8_t* data,
                      const size_t size,
                      StringType (*num_to_string)(NumberType),
                      bool (*string_to_num)(StringPieceType, NumberType*)) {
  // Ensure we can read a NumberType from |data|
  if (size < sizeof(NumberType))
    return;
  const NumberType v1 = *reinterpret_cast<const NumberType*>(data);

  // Because we started with an arbitrary NumberType value, not an arbitrary
  // string, we expect that the function |string_to_num| (e.g. StringToInt) will
  // return true, indicating a perfect conversion.
  NumberType v2;
  CHECK(string_to_num(num_to_string(v1), &v2));

  // Given that this was a perfect conversion, we expect the original NumberType
  // value to equal the newly parsed one.
  CHECK_EQ(v1, v2);
}

template <class NumberType>
void CheckRoundtrips(const uint8_t* data,
                     const size_t size,
                     bool (*string_to_num)(base::StringPiece, NumberType*)) {
  return CheckRoundtripsT<NumberType, base::StringPiece, std::string>(
      data, size, &base::NumberToString, string_to_num);
}

template <class NumberType>
void CheckRoundtrips16(const uint8_t* data,
                       const size_t size,
                       bool (*string_to_num)(base::StringPiece16,
                                             NumberType*)) {
  return CheckRoundtripsT<NumberType, base::StringPiece16, std::u16string>(
      data, size, &base::NumberToString16, string_to_num);
}

// Entry point for LibFuzzer.
extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
  // For each instantiation of NumberToString f and its corresponding StringTo*
  // function g, check that f(g(x)) = x holds for fuzzer-determined values of x.
  CheckRoundtrips<int>(data, size, &base::StringToInt);
  CheckRoundtrips16<int>(data, size, &base::StringToInt);
  CheckRoundtrips<unsigned int>(data, size, &base::StringToUint);
  CheckRoundtrips16<unsigned int>(data, size, &base::StringToUint);
  CheckRoundtrips<int64_t>(data, size, &base::StringToInt64);
  CheckRoundtrips16<int64_t>(data, size, &base::StringToInt64);
  CheckRoundtrips<uint64_t>(data, size, &base::StringToUint64);
  CheckRoundtrips16<uint64_t>(data, size, &base::StringToUint64);
  CheckRoundtrips<size_t>(data, size, &base::StringToSizeT);
  CheckRoundtrips16<size_t>(data, size, &base::StringToSizeT);

  base::StringPiece string_piece_input(reinterpret_cast<const char*>(data),
                                       size);
  std::string string_input(reinterpret_cast<const char*>(data), size);

  int out_int;
  base::StringToInt(string_piece_input, &out_int);
  unsigned out_uint;
  base::StringToUint(string_piece_input, &out_uint);
  int64_t out_int64;
  base::StringToInt64(string_piece_input, &out_int64);
  uint64_t out_uint64;
  base::StringToUint64(string_piece_input, &out_uint64);
  size_t out_size;
  base::StringToSizeT(string_piece_input, &out_size);

  // Test for StringPiece16 if size is even.
  if (size % 2 == 0) {
    base::StringPiece16 string_piece_input16(
        reinterpret_cast<const char16_t*>(data), size / 2);

    base::StringToInt(string_piece_input16, &out_int);
    base::StringToUint(string_piece_input16, &out_uint);
    base::StringToInt64(string_piece_input16, &out_int64);
    base::StringToUint64(string_piece_input16, &out_uint64);
    base::StringToSizeT(string_piece_input16, &out_size);
  }

  double out_double;
  base::StringToDouble(string_input, &out_double);

  base::HexStringToInt(string_piece_input, &out_int);
  base::HexStringToUInt(string_piece_input, &out_uint);
  base::HexStringToInt64(string_piece_input, &out_int64);
  base::HexStringToUInt64(string_piece_input, &out_uint64);
  std::vector<uint8_t> out_bytes;
  base::HexStringToBytes(string_piece_input, &out_bytes);

  base::HexEncode(data, size);

  // Convert the numbers back to strings.
  base::NumberToString(out_int);
  base::NumberToString16(out_int);
  base::NumberToString(out_uint);
  base::NumberToString16(out_uint);
  base::NumberToString(out_int64);
  base::NumberToString16(out_int64);
  base::NumberToString(out_uint64);
  base::NumberToString16(out_uint64);
  base::NumberToString(out_double);
  base::NumberToString16(out_double);

  return 0;
}
```

### windows

```cpp
// base/strings/string_number_conversions_win.cc
namespace base {

std::wstring NumberToWString(int value) {
  return internal::IntToStringT<std::wstring>(value);
}

std::wstring NumberToWString(unsigned value) {
  return internal::IntToStringT<std::wstring>(value);
}

std::wstring NumberToWString(long value) {
  return internal::IntToStringT<std::wstring>(value);
}

std::wstring NumberToWString(unsigned long value) {
  return internal::IntToStringT<std::wstring>(value);
}

std::wstring NumberToWString(long long value) {
  return internal::IntToStringT<std::wstring>(value);
}

std::wstring NumberToWString(unsigned long long value) {
  return internal::IntToStringT<std::wstring>(value);
}

std::wstring NumberToWString(double value) {
  return internal::DoubleToStringT<std::wstring>(value);
}

namespace internal {

template <>
class WhitespaceHelper<wchar_t> {
 public:
  static bool Invoke(wchar_t c) { return 0 != iswspace(c); }
};

}  // namespace internal

bool StringToInt(WStringPiece input, int* output) {
  return internal::StringToIntImpl(input, *output);
}

bool StringToUint(WStringPiece input, unsigned* output) {
  return internal::StringToIntImpl(input, *output);
}

bool StringToInt64(WStringPiece input, int64_t* output) {
  return internal::StringToIntImpl(input, *output);
}

bool StringToUint64(WStringPiece input, uint64_t* output) {
  return internal::StringToIntImpl(input, *output);
}

bool StringToSizeT(WStringPiece input, size_t* output) {
  return internal::StringToIntImpl(input, *output);
}

bool StringToDouble(WStringPiece input, double* output) {
  return internal::StringToDoubleImpl(
      input, reinterpret_cast<const uint16_t*>(input.data()), *output);
}

}  // namespace base
```

