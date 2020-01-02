---
layout:     post
title:      UIViewExt
subtitle:   
date:       2017-01-07
author:     夏天无泪
header-img: img/post-bg-iWatch.jpg
catalog: true
tags:
    - SwiftExt
---

## UIViewExt

**UIView**  

继承自UIResponder，间接继承自NSObject，主要是用来构建用户界面的，并且可以响应事件。对于UIView，侧重于对内容的显示管理；其实是相对于CALayer的高层封装。  

```
**通过UIView对象找到其所在的UIViewController**
extension UIView {
    //返回该view所在VC
    public var parentViewController: UIViewController?{
        weak var parentResponder: UIResponder? = self
        while parentResponder != nil {
            parentResponder = parentResponder!.next
            if let viewController = parentResponder as? UIViewController {
                return viewController
            }
        }
        return nil
    }
        /// 截屏
    public func screenshot() -> UIImage {
        let renderer = UIGraphicsImageRenderer(bounds: bounds)
        return renderer.image { rendererContext in
            layer.render(in: rendererContext.cgContext)
        }
    }

    /// 同时添加多个 subview
    public func addSubviews(_ subviews: UIView...) {
        subviews.forEach(addSubview(_:))
    }
    
    /// 删除视图下的所有 subview
    public func removeSubviews() {
        subviews.forEach({ $0.removeFromSuperview() })
    }

}
```

```
extension UIView {
    /// 开始旋转动画
    public func startRotate(duration: CFTimeInterval = 1.0, completionDelegate: AnyObject? = nil) {
        let rotateAnimation = CABasicAnimation(keyPath: "transform.rotation")
        rotateAnimation.fromValue = 0.0
        rotateAnimation.toValue = CGFloat(Double.pi * 2.0)
        rotateAnimation.duration = duration
        rotateAnimation.repeatCount = MAXFLOAT
        rotateAnimation.isRemovedOnCompletion = false
        
        self.layer.add(rotateAnimation, forKey: "view.animation.rotate")
    }
    
    /// 停止旋转动画
    public func stopRotate() {
        self.layer.removeAnimation(forKey: "view.animation.rotate")
    }
}
```  

```
extension UIView {
    public enum presentStyle: Int {
        case center, bottom
    }
    
    /// 自定义视图弹出效果支持从中间出来和从底下出来，背景视图支持点击消失
    public func present(_ from: presentStyle = .center, tapBackgroundClose: Bool = false, useSafeArea: Bool = false, completionHandler: @escaping () -> Void = {}) {
        guard let window = UIApplication.shared.windows.first else { return }
        
        window.endEditing(true)
        
        self.tag = 1010122
        
        let backgroundView = UIView(frame: window.bounds)
        backgroundView.backgroundColor = UIColor.black
        backgroundView.alpha = 0.0
        backgroundView.tag = 1010123

        window.subviews.forEach { [weak self] view in // 避免多次 addSubview
            if view.className == self?.className {
                view.removeFromSuperview()
                window.viewWithTag(1010123)?.removeFromSuperview()
            }
        }

        window.addSubview(backgroundView)
        window.addSubview(self)

        RxKeyboard.instance.visibleHeight.drive(onNext: { [weak self] keyboardVisibleHeight in
            self?.y = (Device.height - keyboardVisibleHeight - (self?.height ?? 0)) / 2
        }).disposed(by: rx.disposeBag)
        
        if tapBackgroundClose {
            backgroundView.rx.tapGesture().when(.recognized).subscribe(onNext: { [weak self] _ in
                self?.dismiss()
            }).disposed(by: rx.disposeBag)
        }
        
        if from == .center {
            self.alpha = 0.0
            self.transform = CGAffineTransform(scaleX: 0.1, y: 0.1)
            
            UIView.animate(withDuration: 0.3, animations: {
                backgroundView.alpha = 0.4
                self.alpha = 1
                self.transform = CGAffineTransform(scaleX: 1.0, y: 1.0)
            }, completion: { (succes) in
                completionHandler()
            })
        } else {
            self.y = Device.height
            self.alpha = 1
            
            var newY = Device.height - self.height
            
            if useSafeArea, #available(iOS 11.0, *) {
                newY -= self.safeAreaInsets.bottom
            }
            
            UIView.animate(withDuration: 0.3, animations: {
                backgroundView.alpha = 0.4
                
                self.y = newY
            }, completion: { (succes) in
                completionHandler()
            })
        }
    }
    
    /// 移除自定义视图
    public func dismiss(_ completionHandler: @escaping () -> Void = {}) {
        guard  let window = UIApplication.shared.windows.first else { return }
        
        window.endEditing(true)
        
        var backgroundView: UIView?
        var currentView: UIView?
        
        // 如果 window 上同时加了多个自定义视图，则从最后一个开始移除
        window.subviews.forEach { subview in
            if subview.tag == 1010123 {
                backgroundView = subview
            }
            if subview.tag == 1010122 {
                currentView = subview
            }
        }
        
        UIView.animate(withDuration: 0.3, animations: {
            backgroundView?.alpha = 0
            currentView?.alpha = 0
        }, completion: { completed in
            backgroundView?.removeFromSuperview()
            currentView?.removeFromSuperview()
            
            completionHandler()
        })
    }
}

/// view 渐变色效果
typealias GradientPoints = (startPoint: CGPoint, endPoint: CGPoint)

public enum GradientOrientation {
    case topRightBottomLeft
    case topLeftBottomRight
    case horizontal
    case vertical
    
    var startPoint: CGPoint {
        return points.startPoint
    }
    
    var endPoint: CGPoint {
        return points.endPoint
    }
    
    var points: GradientPoints {
        switch self {
        case .topRightBottomLeft:
            return (CGPoint(x: 0.0, y: 1.0), CGPoint(x: 1.0, y: 0.0))
        case .topLeftBottomRight:
            return (CGPoint(x: 0.0, y: 0.0), CGPoint(x: 1, y: 1))
        case .horizontal:
            return (CGPoint(x: 0.0, y: 0.5), CGPoint(x: 1.0, y: 0.5))
        case .vertical:
            return (CGPoint(x: 0.0, y: 0.0), CGPoint(x: 0.0, y: 1.0))
        }
    }
}

extension UIView {
    public func applyGradient(withColours colours: [UIColor], gradientOrientation orientation: GradientOrientation) {
        for layer in self.layer.sublayers ?? [] where layer.className == "CAGradientLayer"{
            layer.removeFromSuperlayer()
        }
        
        let gradient: CAGradientLayer = CAGradientLayer()
        gradient.frame = self.bounds
        gradient.colors = colours.map { $0.cgColor }
        gradient.startPoint = orientation.startPoint
        gradient.endPoint = orientation.endPoint
        gradient.locations = nil
        self.layer.insertSublayer(gradient, at: 0)
    }
}
```



##**待补充**