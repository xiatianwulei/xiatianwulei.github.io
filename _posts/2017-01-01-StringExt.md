---
layout:     post
title:      StringExt
subtitle:   
date:       2017-01-06
author:     夏天无泪
header-img: img/post-bg-iWatch.jpg
catalog: true
tags:
    - SwiftExt
---
# swiftExt  

Swift语言的类扩展是一个强大的工具，我们可以通过类扩展完成如下事情：  
1. 给已有的类添加计算属性和计算静态属性  
2. 定义新的实例方法和类方法   
3. 提供新的构造器   
4. 定义下标脚本   
5. 是一个已有的类型符合某个协议   


```
extension String
{
	/// 给字符串String类添加下标脚本，支持索引访问  
    subscript(start:Int, length:Int) -> String
    {
        get{
            let index1 = self.index(self.startIndex, offsetBy: start)
            let index2 = self.index(index1, offsetBy: length)
            return String(self[index1..<index2])
        }
        set{
            let tmp = self
            var s = ""
            var e = ""
            for (idx, item) in tmp.characters.enumerated() {
                if(idx < start)
                {
                    s += "\(item)"
                }
                if(idx >= start + length)
                {
                    e += "\(item)"
                }
            }
            self = s + newValue + e
        }
    }
    subscript(index:Int) -> String
    {
        get{
            return String(self[self.index(self.startIndex, offsetBy: index)])
        }
        set{
            let tmp = self
            self = ""
            for (idx, item) in tmp.characters.enumerated() {
                if idx == index {
                    self += "\(newValue)"
                }else{
                    self += "\(item)"
                }
            }
        }
    }
}
 
```  

```   
extension String {
    /// AES 加密后的字符串：ECB / pkcs7 / iv 不偏移
    public var toAESString: String {
        if let cipher = try? AES(key: App.aesKey.bytes, blockMode: ECB()).encrypt(self.bytes) {
            return cipher.toHexString()
        }
        
        return ""
    }
    
    /// 将URL中的"特殊字符"转成“%3A%2F%2F”
    public var urlEncoded: String {
        return addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed)!
    }

    /// 将URL中的“%3A%2F%2F”转成正常字符
    public var urlDecode: String {
        return removingPercentEncoding!
    }

    /// 是否是手机号
    public var isPhoneNumber: Bool {
        return self.hasPrefix("1") && self.count == 11 && Int64(self) != nil
    }

    /// 136****3119, 星号也可设置成别的字符
    public func cover(_ separator: String = "****") -> String {
        guard self.isPhoneNumber else { return self }
        
        return (self as NSString).replacingCharacters(in: NSRange(location: 3, length: 4), with: separator)
    }
    
    /// 136 9134 3119, 空格也可设置成别的
    public func format(_ separator: String = " ") -> String {
        guard self.isPhoneNumber else { return self }
        
        let index1 = self.index(self.startIndex, offsetBy: 3)
        let index2 = self.index(index1, offsetBy: 4)
        
        return "\(self[self.startIndex..<index1])" + separator + "\(self[index1..<index2])" + separator + "\(self[index2..<self.endIndex])"
    }
}
```  

```
extension String {
    /// 转成 Date，样式可修改
    public func date(withFormat format: String = "yyyy-MM-dd HH:mm") -> Date? {
        let dateFormatter = Date.formatter
        dateFormatter.dateFormat = format
        return dateFormatter.date(from: self)
    }
    
    /// 转成 JSON 对象
    public func toJSON() -> Any? {
        do {
            return try JSON(data: data(using: .utf8)!).object
        } catch {
            return nil
        }
    }
    
    /// 返回去掉首尾空格和新行的字符串
    public func trim() -> String {
        return self.trimmingCharacters(in: .whitespacesAndNewlines)
    }
    
    /// 按字数裁断字符串，用 ... 表示剩余部分
    public func truncated(toLength length: Int, trailing: String? = "...") -> String {
        guard 1..<count ~= length else { return self }
        return self[startIndex..<index(startIndex, offsetBy: length)] + (trailing ?? "")
    }
    
    /// 给字符串添加下划线
    public var underline: NSAttributedString {
        return NSAttributedString(string: self, attributes: [.underlineStyle: NSUnderlineStyle.single.rawValue])
    }
    
    /// 给字符串添加删除线
    public var strikethrough: NSAttributedString {
        return NSAttributedString(string: self, attributes: [.strikethroughStyle: NSNumber(value: NSUnderlineStyle.single.rawValue as Int)])
    }
    
    /// 给字符串添加固定的行间距
    public func addLineSpacing(_ spacing: CGFloat = 5, fontSize: CGFloat = 17) -> NSAttributedString {
        let paragraphStyle = NSMutableParagraphStyle()
        paragraphStyle.lineSpacing = spacing
        
        return NSAttributedString(string: self, attributes: [.font: UIFont.systemFont(ofSize: fontSize), .paragraphStyle: paragraphStyle])
    }
    /// 计算文本的size
    ///
    /// - Parameters:
    ///   - constrainedSize: 限制的size
    ///   - font: 字号
    ///   - lineSpacing: 默认为nil，使用系统默认的行间距
    /// - Returns: 计算文本的size
    public func stringBoundingRect(with constrainedSize: CGSize, font: UIFont, lineSpacing: CGFloat? = nil) -> CGSize {
        let attritube = NSMutableAttributedString(string: self)
        let range = NSRange(location: 0, length: attritube.length)
        attritube.addAttributes([NSAttributedString.Key.font: font], range: range)
        if lineSpacing != nil {
            let paragraphStyle = NSMutableParagraphStyle()
            paragraphStyle.lineSpacing = lineSpacing!
            attritube.addAttribute(NSAttributedString.Key.paragraphStyle, value: paragraphStyle, range: range)
        }
        let rect = attritube.boundingRect(with: constrainedSize, options: [.usesLineFragmentOrigin, .usesFontLeading], context: nil)
        var size = rect.size
        
        if let currentLineSpacing = lineSpacing {
            // 文本的高度减去字体高度小于等于行间距，判断为当前只有1行
            let spacing = size.height - font.lineHeight
            if spacing <= currentLineSpacing && spacing > 0 {
                size = CGSize(width: size.width, height: font.lineHeight)
            }
        }
        
        return size
    }
    /// 计算文本的size 可以限制行数
    public func stringBoundingRect(with constrainedSize: CGSize, font: UIFont, lineSpacing: CGFloat? = nil, lines: Int) -> CGSize {
        if lines < 0 {
            return .zero
        }
        
        let size = stringBoundingRect(with: constrainedSize, font: font, lineSpacing: lineSpacing)
        if lines == 0 {
            return size
        }
        
        let currentLineSpacing = (lineSpacing == nil) ? (font.lineHeight - font.pointSize) : lineSpacing!
        let maximumHeight = font.lineHeight * CGFloat(lines) + currentLineSpacing * CGFloat(lines - 1)
        if size.height >= maximumHeight {
            return CGSize(width: size.width, height: maximumHeight)
        }
        
        return size
    }
    
    /// base64 编码
    var base64Encoded: String? {
        let plainData = data(using: .utf8)
        return plainData?.base64EncodedString()
    }
    
    /// base64 解码
    var base64Decoded: String? {
        guard let decodedData = Data(base64Encoded: self) else { return nil }
        return String(data: decodedData, encoding: .utf8)
    }
    
    /// 是否包含字符
    var hasLetters: Bool {
        return rangeOfCharacter(from: .letters, options: .numeric, range: nil) != nil
    }
    
    /// 是否包含数字
    var hasNumbers: Bool {
        return rangeOfCharacter(from: .decimalDigits, options: .literal, range: nil) != nil
    }
}
```

```
/// 实现NSRange与Range的相互转换
extension String {
     
    //Range转换为NSRange
    func nsRange(from range: Range<String.Index>) -> NSRange {
        let from = range.lowerBound.samePosition(in: utf16)
        let to = range.upperBound.samePosition(in: utf16)
        return NSRange(location: utf16.distance(from: utf16.startIndex, to: from),
                       length: utf16.distance(from: from, to: to))
    }
     
    //Range转换为NSRange
    func range(from nsRange: NSRange) -> Range<String.Index>? {
        guard
            let from16 = utf16.index(utf16.startIndex, offsetBy: nsRange.location,
                                     limitedBy: utf16.endIndex),
            let to16 = utf16.index(from16, offsetBy: nsRange.length,
                                   limitedBy: utf16.endIndex),
            let from = String.Index(from16, within: self),
            let to = String.Index(to16, within: self)
            else { return nil }
        return from ..< to
    }
}
```

### **待补充**
