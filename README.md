# Crucible

Crucible is a platform for loading C function or Objective-C method hooks from plists.

## Installation

Crucible comes bundled with Parasite on installation.

## Folder Structure

Place Crucible plist files in `/Library/Parasite/Crucible`. You can name them using the following options:

* File has the name `class_<CLASSNAME>` where CLASSNAME is whatever class you want to trigger the loading of the plist if it is present in the target process.
* File has the same name as the bundle identifier it is targeting.
* File has any name, inside of folder with any of the above mentioned filters.

## Format

Crucible files are just in a certain `plist` file format. Here is the general structure you want and an explanation of the keys.


```
| Root (plist root)
|––––| Hooks (array)
|––––|––––| Index 0 (dictionary)
|––––|––––|––––| Key: Class (string): "NSColor"
|––––|––––|––––| Key: Methods (array)
|––––|––––|––––|––––| Index 0 (array)
|––––|––––|––––|––––|––––| Index 0 (string): colorWithSRGBRed:green:blue:alpha:
|––––|––––|––––|––––|––––| Index 1 (any): dictionary
|––––|––––|––––|––––|––––|––––| Key: Value (value element): "#FFF"
|––––|––––|––––|––––|––––|––––| Key: Transform (string): "color"
|––––|––––|––––|––––| Index 1 (array)
|––––|––––|––––|––––|––––| Index 0 (string): "+description"
|––––|––––|––––|––––|––––| Index 1 (any): "Hello, Class Description"
|––––|––––|––––|––––| Index 2 (array)
|––––|––––|––––|––––|––––| Index 0 (string): "-description"
|––––|––––|––––|––––|––––| Index 1 (any): "Hello, Instance Description"
|––––|––––| Index 1 (dictionary)
|––––|––––|––––| Key: MinBundleVersion (number): 0
|––––|––––|––––| Key: MaxBundleVersion (number): 999
|––––|––––|––––| Key: Symbol (string): "_NSUserName"
|––––|––––|––––| Key: Image (string): "identifier:com.apple.AppKit"
|––––|––––|––––| Key: Returns (string): "@"
|––––|––––|––––| Key: Value (value/value dict): "Bill Nye"
|––––|––––| Index 2 (dictionary)
|––––|––––|––––| Key: Symbol (string): "_test"
|––––|––––|––––| Key: Image (string): "@executable_path"
|––––|––––|––––| Key: Returns (string): "v"
|––––|––––|––––| Key: Value (value/value dict): ""
```

### Description

| Name | Type | Description |
|-----|------|-------------|
| Hooks | array of Hook Elements | The root key that contains all of your hooks |
| Hook Element | dictionary | Contains a **class hook** or a **function hook** |
| MinBundleVersion | number | Minimum Bundle Version to use this `Hook Element` on. The version is derived from whatever filter you used to load this Crucible plist. |
| MaxBundleVersion | number | Maximum Bundle Version to use this `Hook Element` on. |
| Class | string | For a **class hook**, specifies which class the `Methods` key will hook |
| Methods | array of Method Elements | For a **class hook**, contains a list of all the methods in the class to hook and which values to return. |
| Method Element | array | For a **class hook**, describes a method to hook and the return value. The ***first*** value is the selector of the method to replace; you can optionally prefix with +/- to disambiguate class and instance methods, but leaving it off lets Crucible check both. The ***second*** element is a *Value Element* of the return value to use. |
| Symbol | string | For a **function hook**, Public/Private symbol name that you would like to hook |
| Image | string | For a **function hook**, Path to a binary where we can find your function. You can prefix this with `identifier:` to have Crucible dynamically find the executable given a bundle identifier, or you can write `@executable_path` to specify the Application binary path. |
| Returns | string | For a **function hook**, a type encoding of the return type of the function you are hooking. [More Information](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)
| Value | value element | Value to return |
| Value Element | dictionary/any | Anywhere you are writing something as a return value, whether inside a function hook or the method hook, your objects will be recursively `sanitized`. This is just a fancy word we used to say that whenever I see a dictionary with the key `Value` Crucible will treat that sub-element as another *Value Element* and collapse the original one into just the smaller one. Why would you do this? So you can also have a `Transform` key and apply transforms to values in dictionaries and arrays. |

### Transforms

Transforms take whatever value you entered and does some pre-processing at runtime to make make it a different value. 

Parasite extensions can add more Transforms into whatever they are loaded into by defining a category on `NSObject` and naming functions with a `_crucible_` prefix such as `-(id)_crucible_url` for the `url` transform. ***NOTE:*** Transforms are affected by any hooks specified in parasite or Crucible, so be wary of infinite loops with things like for example; when hooking `colorWithRed:green:blue:alpha:` and supplying an argument that would trigger an invocation of that function because you will be stuck in an infinite loop.

Here is a list of the default available transforms:

| Name | Return Type | Argument Type | Description |
|------|-------------|---------------|-------------|
| url  | NSURL | string | Transforms a string into an NSURL using `URLWithString:` |
| file_url | NSURL | string | Transforms a string into a file path URL using `fileURLWithPath:` |
| now | NSDate | ignored | Returns `[NSDate date]` |
| color | NSColor | string/number | Transforms a string or a number into an NSColor. Accepts the common CSS styles like hsl/hsla, hsb, rgb, gray, hex. |
| alloc_init | any class | string | Calls +alloc and then -init on a given class name.
