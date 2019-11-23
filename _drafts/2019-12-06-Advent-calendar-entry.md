---
layout: post
date: 2019-11-17 08:00:00 +0100
categories: fsharp fsi interactive
---

This post is an entry in the [2019 F# advent calendar](https://sergeytihon.com/2019/11/05/f-advent-calendar-in-english-2019/) hosted by Sergey Tihon.

### Disclaimer

These experiences are driven by my day job where I work a lot with `DateTime` and such.

> This is some long
> and fancy quote
> that must be here

### links
[fsi tricks](https://blogs.msdn.microsoft.com/dsyme/2010/01/08/f-interactive-tips-and-tricks-formatting-data-using-addprinter-addprinttransformer-and-a-in-sprintfprintffprintf/)

### F# interactive

[fsi.exe](https://docs.microsoft.com/en-us/dotnet/fsharp/tutorials/fsharp-interactive/) is a tool for interactive development in F#. It comes with Visual Studio when F# tools are installed and is available in [VSCode](https://code.visualstudio.com/) with the [Ionide](http://ionide.io/) plugin.

It is extremely useful when exploring .NET libraries, experimenting with the implementation of a new feature and as a general scratchpad for your code. In any `*.fs` or `*.fsx` file simply highlight a section of code to execute and hit `Alt + Enter` and the result will become available in the interactive console window.

The output is usually formatted quite nicely. Take this custom record type:

```fsharp
type Person = 
  { FirstName   : string
    LastName    : string
    DateOfBirth : DateTime }
```

When a new instance of `Person` is declared the formatting is pretty good:

```fsharp
let person =
  { FirstName   = "John"
    LastName    = "Doe"
    DateOfBirth = DateTime(1990, 06, 12, 08, 03, 33, DateTimeKind.Local) }

val person : Person = { FirstName = "John"
                        LastName = "Doe"
                        DateOfBirth = 1980-06-12 12.03.33 }
```

However, if we print the newly created value again the out is quite different:

```fsharp
> person;;
val it : Person =
  { FirstName = "John"
    LastName = "Doe"
    DateOfBirth = 1980-06-12 12.03.33 {Date = 1980-06-12 00.00.00;
                                       Day = 12;
                                       DayOfWeek = Thursday;
                                       DayOfYear = 164;
                                       Hour = 12;
                                       Kind = Local;
                                       Millisecond = 0;
                                       Minute = 3;
                                       Month = 6;
                                       Second = 33;
                                       Ticks = 624652562130000000L;
                                       TimeOfDay = 12:03:33;
                                       Year = 1980;} }
```
Similar verbose output is observed for `Uri`, `TimeSpan` and `DateTimeOffset`. Enums do not look too great either, as this example shows:
```fsharp
> DayOfWeek.Monday;;
val it : DayOfWeek = Monday {value__ = 1;}
```
This becomes less than ideal when working interactively with these types in the long run. Luckily there is a solution.

### Print transformers
In `*.fsx` files there is a globally available value named `fsi` with a few methods and values that can be used to modify the behavior of the current interactive session. Here I will discuss the method `fsi.AddPrinter('a -> string)`.

It takes a callback that takes a value of type `'a` and returns a string which will then be printed as is.

Here is an how to register an improved `DateTime` printer:
```fsharp
fsi.AddPrinter<DateTime>(fun dt -> dt.ToString("s"))
```
Whenever a `DateTime` value is printed to the interactive console it will be nicely formatted. In addition any `DateTime` properties of existing types will also use this formatter. Our `person` from before now renders like this:
```fsharp
> person;;
val it : Person = { FirstName = "John"
                    LastName = "Doe"
                    DateOfBirth = 1980-06-12T12:03:33 }
```
I like to format my `DateTime`s a little more elaborately:
```fsharp
fsi.AddPrinter<DateTime>(fun dt ->
  let time = dt.TimeOfDay
  let format =
    if   time.Ticks        = 0L then "yyyy-MM-dd"
    elif time.Seconds      = 0  then "yyyy-MM-dd HH:mm"
    elif time.Milliseconds = 0  then "yyyy-MM-dd HH:mm:ss"
                                else "yyyy-MM-dd HH:mm:ss.fff"
  let str = dt.ToString(format, CultureInfo.InvariantCulture)
  sprintf "%s (%O)" str dt.Kind)
```
This formatting has two benefits. First, the format is as short as possible. If the `DateTime` value has no time component, only the date is shown. Likewise, second and millisecond components are only printed if they are non-zero. Second, the `DateTimeKind` property is explicitly printed. I find this very useful when debugging code that contains `DateTime` values, the `Kind` property is notoriously cumbersome to work with (link). 

Here are all my printers

```fsharp
fsi.AddPrinter<Uri>(fun x -> x.ToString())
fsi.AddPrinter<Enum>(fun x -> x.GetType().Name + "." +  x.ToString())
fsi.AddPrinter<TimeSpan>(fun x -> x.ToString())
fsi.AddPrinter<DateTimeOffset>(fun x -> x.ToString())
fsi.AddPrinter<FileInfo>(fun x -> x.FullName)
fsi.AddPrinter<DirectoryInfo>(fun x -> x.FullName)
```

Note how `Enum`s are prefixed with type name for extra readability.

### Always on
It would be tedious to copy this code in to every new `*.fsx` file. I like to put these *settings* in a central file called `prelude.fsx` ([from Haskell](https://wiki.haskell.org/Prelude))

In order to load the printers in every session the `--use` directive can be specified when starting `fsi.exe`.

In Visual Studio go to `Tools > Options > F# Tools` and pass extra parameters like this:
![Visual Studio Prelude](/assets/path_to_prelude_visual_studio.png "Visual Studio Prelude")

In Visual Studio Code, go to `File > Preferences > Settings` or hit `Ctrl+,`, search for `Fsi Extra Parameters`, click `Edit in settings.json`:

![Ionide settings](/assets/ionide_settings.png "Visual Studio Prelude")

and add the following:
```json
"FSharp.fsiExtraParameters": [
    "--use:C:\\path\\to\\prelude.fsx"
]
```

### Final bits
For completeness, here are additional helper functions that I've added to my `prelude.fsx` which come in very handy from time to time. Note that I have only tested these on Windows

```fsharp
open System
open System.IO
open System.Diagnostics
open System.Globalization

/// Start a new process for the path.
let run path = Process.Start(fileName = path) |> ignore

/// Open this file for editing.
let editPrelude() =
  let scriptPath = Path.Combine(__SOURCE_DIRECTORY__, __SOURCE_FILE__)
  printfn "Opening %s for editing." scriptPath
  run scriptPath

/// Copies the item into the clipboard formatted as a string.
let clip obj =
  let text =
    match box obj with
    | :? String as s -> s
    | :? DateTime as s ->
      s.ToString("yyyy-MM-dd HH:mm:ss.fff", CultureInfo.InvariantCulture)
    | _ -> sprintf "%A" obj
  if text = "" then
    printfn "No content, clipboard remains unchanged"
  else
    Windows.Forms.Clipboard.SetText(text, Windows.Forms.TextDataFormat.Text)
    printfn "Copied %i characters to clip board" text.Length
  obj
```
