# UTF8-多字节变长存储

# ASCII

计算机是美国人发明的，由于他们的语言的是美式英语，字符比较少，所以一开始就设计了一个不大的二维表，128个字符，取名**ASCII**（American Standard Code for Information Interchange）。128个码位（包括32个不能打印出来的控制符号）只占用了一个字节的后面7位，最前面的1位统一规定为0，其表示范围为`00000000-01111111`或`0x00-0x7F`。

[![img](http://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/blog-images/ascii.gif)](http://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/blog-images/ascii.gif)

后来美国人发现128个字符不够用，于是在原来的二维表的基础上进行了扩展，利用ASCII编码字节中闲置的最高位，从而编入新的符号。这种扩展后的编码被称为**EASCII**（Extended ASCII）。256个码位正好可以使用一个字节表示，其表示范围为`00000000-11111111`或`0x00-0xFF`。

[![img](http://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/blog-images/eascii.gif)](http://chuquan-public-r-001.oss-cn-shanghai.aliyuncs.com/blog-images/eascii.gif)

当计算机技术传递到欧洲后，美国的编码标准就不适用了，但是改改还能凑合。于是国际标准化组织在ASCII的基础上进行了扩展，形成了ISO-8859标准，跟EASCII类似，兼容ASCII，在高128个码位上有所区别。但是由于欧洲的语言环境十分复杂，所以根据各地区的语言又形成了很多子标准，如ISO-8859-1、ISO-8859-2、ISO-8859-3，……，ISO-8859-16等等。

# 非ASCII

单个字节表示256个字符的ASCII系编码，对于欧洲各国的语言尚可表示，然而，对于亚洲地区的语言来说则远远不够，中国的汉字就多达10万左右。因此，就必须使用多个字节表示一个字符。为此在，亚洲地区又出现了很多编码，大陆的GB2312和GBK（GB2312的扩展，Kuozhan）、港台的BIG5、日本的Shift JIS等等。例如，GB2312编码使用两个字节表示一个汉字，所以理论上最多可以表示65536个汉字。

# Unicode

当互联网席卷了全球，地域限制被打破了，不同国家和地区的计算机在交换数据的过程中，一旦使用不同的字符编码就会出现乱码的问题。要彻底解决这个问题，就必须使用一个通用的字符集**UCS**（Universal Character Set）和一个通用的字符编码**Unicode**。

可以想象，如果有一种编码，将世界上所有的符号都纳入其中。每一个符号都给予一个独一无二的编码，那么就可以彻底解决乱码问题。这就是Unicode，就像它的名字一样，这是一种所有符号的编码。

Unicode是一个很大的集合，现在的规模可以容纳100多万个符号。每个符号的编码都不一样，比如，U+0639表示阿拉伯字母Ain，U+0041表示英语的大写字母A，U+4E25表示汉字”严”。具体的符号对应表，可以查询[unicode.org](http://www.unicode.org/)，或者专门的[汉字对应表](http://www.chi2ko.com/tool/CJK.htm)。

> Unicode只是一个符号集，只规定了符号的二进制代码，却没有规定这个二进制代码应该如何存储。

## 问题

事实上，Unicode在实现上存在两个需要解决的问题：

> (1) 如果采用多字节定长存储，则面临着极大的存储空间浪费以及字符集扩展问题。如：单字节的ASCII编码将浪费大量存储空间。
> (2) 如果采用多字节非定长存储，则面临着不定长编码的识别问题。即，如何知道一个编码使用了多少字节。

目前，有三种被广泛认知的Unicode实现方式：**UTF-8**，**UTF-16**（字符用两个字节或四个字节表示），**UTF-32**（字符用四个字节表示）。然而只有UTF-8有效地解决了上述两个问题，使得其成为目前最广泛的Unicode实现方式。

## UTF-8

UTF-8最大的一个特点，就是它是一种变长的编码方式。它可以使用1~4个字节表示一个符号，根据不同的符号而变化字节长度。

UTF-8的编码规则很简单，只有二条：

> (1) 对于单字节的符号，字节的第一位设为0，后面7位为这个符号的unicode码。因此对于英语字母，UTF-8编码和ASCII码是相同的。
> (2) 对于n字节的符号（n>1），第一个字节的前n位都设为1，第n+1位设为0，后面字节的前两位一律设为10。剩下的没有提及的二进制位，全部为这个符号的unicode码。

下表总结了UTF-8编码规则，字母x表示可用编码的位。

| Unicode符号范围(十六进制) | UTF-8编码方式(二进制)               |
| :------------------------ | :---------------------------------- |
| 0000-0000-0000-007F       | 0xxxxxxx                            |
| 0000-0080-0000-07FF       | 110xxxxx 10xxxxxx                   |
| 0000-0800-0000-FFFF       | 1110xxxx 10xxxxxx 10xxxxxx          |
| 0001-0000-0010-FFFF       | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx |

UTF-8的编码规则：如果第一个字节的第一位是0，则这个字节单独就是一个字符；如果第一位是1，则连续有多少个1，就表示当前字符占用多少个字符。

下面，还是以汉字”严”为例，演示如何实现UTF-8编码。
已知”严”的unicode是4E25（100111000100101），根据上表，可以发现4E25处在第三行的范围内（0000 0800-0000 FFFF），因此”严”的UTF-8编码需要三个字节，即格式是”1110xxxx 10xxxxxx 10xxxxxx”。然后，从”严”的最后一个二进制位开始，依次从后向前填入格式中的x，多出的位补0。这样就得到了，”严”的UTF-8编码是”11100100 10111000 10100101”，转换成十六进制就是E4B8A5。