# Dismantling of UEFI HII Dataabse (Part 1)

## Background
On most systems, you can enter into a system BIOS menu by a certain kestroke, interrupting the normal flow and the system presenting you with a set of navigatable forms. On UEFI based firmware, the mechanism through which this data is interacted with is part of Human Interface Infrastructure. Typically, this data is packed in .hpk extensions and the database format is standardized. 

## String Representation in HII Database
If you look at the implementation of a driver that has to present configuration items in the BIOS menu, typically there is an associated .uni extension file. This ends up in a string _package_. Typically, a string package is a bunch of strings with a unique identifier local to package entry. 

As having introduced _package_ in previous sentence, let's briefly idenitfy what a _package_ is: HII database is organized in the form of packages of various types. For exhaustive list, it is recommended to go through section 33.3 of [UEFI Specs](https://uefi.org/sites/default/files/resources/UEFI_Spec_2_10_Aug29.pdf). However, in Tianocore implementation, the most important types are identifed in `UEFIInternalFormRepresentation.h` in Mde package in the form of pre-processor directives.
```C
#define EFI_HII_PACKAGE_TYPE_ALL           0x00
#define EFI_HII_PACKAGE_TYPE_GUID          0x01
#define EFI_HII_PACKAGE_FORMS              0x02
#define EFI_HII_PACKAGE_STRINGS            0x04
#define EFI_HII_PACKAGE_FONTS              0x05
#define EFI_HII_PACKAGE_IMAGES             0x06
#define EFI_HII_PACKAGE_SIMPLE_FONTS       0x07
#define EFI_HII_PACKAGE_DEVICE_PATH        0x08
#define EFI_HII_PACKAGE_KEYBOARD_LAYOUT    0x09
#define EFI_HII_PACKAGE_ANIMATIONS         0x0A
#define EFI_HII_PACKAGE_END                0xDF
#define EFI_HII_PACKAGE_TYPE_SYSTEM_BEGIN  0xE0
#define EFI_HII_PACKAGE_TYPE_SYSTEM_END    0xFF
```
Coming back to string package, now we know a string package will be identified by `EFI_HII_PACKAGE_STRINGS`. For a generic package header, we take help of `UEFIInternalFormRepresentation.h` again:
```C
typedef struct {
  UINT32    Length : 24;
  UINT32    Type   : 8;
  // UINT8  Data[...];
} EFI_HII_PACKAGE_HEADER;
```
Here `Type` identifies the type of package and `Length` describes the total length of the pacakge in bytes. `Type` here dictates how the following data is packed. Looking at `UEFIInternalFormRepresentation.h`, string header is defined as:

```C
typedef struct _EFI_HII_STRING_PACKAGE_HDR {
  EFI_HII_PACKAGE_HEADER    Header;
  UINT32                    HdrSize;
  UINT32                    StringInfoOffset;
  CHAR16                    LanguageWindow[16];
  EFI_STRING_ID             LanguageName;
  CHAR8                     Language[1];
} EFI_HII_STRING_PACKAGE_HDR;
```
As you would have noticed, the first entry of `EFI_HII_STRING_PACKAGE_HDR` is `EFI_HII_PACKAGE_HEADER` which we described earlier. In essence, based on `Type` field, our header is _morphed_ into a type of specific header type, exposing more information related to the type of header. The last field of header describes the language of the following unicode string block in 'ASCII'.

Typically, a _package_ contains further blocks of data packed; similarly a string block will have further string blocks. The string is encoded in unicode format i-e 2 bytes per character. For now, we will look at following two important string blocks:
```C
#define EFI_HII_SIBT_STRING_UCS2        0x14
#define EFI_HII_SIBT_END                0x00
```
`EFI_HII_SIBT_END` is the easiest, as it signifies end of the block. `EFI_HII_SIBT_STRING_UCS2` block contains a string of arbitrary length encoded in unicode format. Thhe corresponding block looks like the following:
```C
typedef struct _EFI_HII_SIBT_STRING_UCS2_BLOCK {
  EFI_HII_STRING_BLOCK    Header;
  CHAR16                  StringText[1];
} EFI_HII_SIBT_STRING_UCS2_BLOCK;
```
where `Header=EFI_HII_SIBT_STRING_UCS2`.

If you refer back to my comment of string package having a unique identifier, you will notice that there is no such identifier associated with the strings here in the header. This unique identifier is basically a simple counter starts with a value of 1 from the header and increments by 1 for each such a block. Coming back to .uni files, each such file basically ends up in string package and is referred to by the unique identifier.




Another package type is _form_ packages as specified by pre-processor directives. Forms govern how the data is shown to the user, things such as navigation of BIOS menu, options presented to user and in a way there are presented (checkbox, dropdown etc) all are specified in Form package.

Before we take a look into _form_ package, let's first unsderstand the direct approach as to how a driver's writer would introduce configurable items into the BIOS menu. 
_Disclaimer: What follows belows is just *a* single way of implementation, described here to provide an overall overview of the process._

## How a driver within UEFI framework might publish BIOS Setup Menu Options?

A driver's writer might introduce _non-volatile_ variables, the values of which are configurable by the user and are influecning the bootflow somehow. 
What I refer to variable here is referred to as value in UEFI specs. A variable according to UEFI spec is a contiguous memory which might contain thousands of bytes. A value is at an offset within _variable_, made up of a couple of bytes. The _values_ might influence the bootflow, especially how a driver initializes a certain functionality. In order it be user configurable, it needs to be presented somewhere in the forms. For that a driver's writer might add a Visual Forms Representation (VFR) file. A .vfr file will contain information on how the options will be presented in front of the user (i-e a checkbox, a question, a dropdown, help, prompt), where will they appear (where in the navigation of BIOS menu), and what will be influenced (value ,restart needed or not or/and callbacks). VFR is represnted in its own syntax, which is defined [here](https://tianocore-docs.github.io/edk2-VfrSpecification/release-1.92/edk2-VfrSpecification-release-1.92.pdf). VFR employs _VFRCompile_ utility to generate Internal Forms Representation (IFR), which consists of representation of the forms in binary format. It produces a file with .hpk extension, which can be parsed with several tools, one of such tool is [IfrViewer](https://github.com/topeterk/IfrViewer).
A .vfr file contains only the string IDs, associated varstores(variable) and variables (values). The strings are contained in a seperate file with an extension ".uni". The strings are then packed seperately in .hpk (in the format we already have touched upon above). These package entries (.hpk) (e.g _form_ package and string package) are then packed to form a package list. We will come back to this detail, when we decompile an HII database. 

To put things together, a driver might introduce a configurable value (variable) influencing certain elements of bootflow. This value is then published in a varstore (variable) i-e a contiguous space identified by a GUID, the corresponding options, associated with the value and to be presented in BIOS menu, are then described in the form of VFR syntax and publisehd through an HII protocol in HII DB. The change in the configuration item then can invoke certain actions. The typical action for instance could be to reset the target, so that upon next installation of driver related protocols (or later), the driver can utilize the value to influence user configurable items in its logic.

## Coming up Next
In Part *2* we will look into:
1.  _FORM_ packages. 
2. Writing a UEFI shell application decompiling HII DB to retrieve BIOS menu options and associated strings
