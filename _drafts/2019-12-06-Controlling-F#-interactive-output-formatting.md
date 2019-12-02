---
layout: post
date: 2019-12-02 08:00:00 +0100
categories: programming fsharp fsi interactive
---

This post is an entry in the [2019 F# advent calendar](https://sergeytihon.com/2019/11/05/f-advent-calendar-in-english-2019/) hosted by Sergey Tihon.

### Intro

A great deal of my daily development work concerns integrating with internal and 3rd party APIs. To that end I find the interactive development provided by `F#` and `fsi` ([F# Interactive](https://docs.microsoft.com/en-us/dotnet/fsharp/tutorials/fsharp-interactive/){:target="_blank"}) extremely helpful when figuring out how the APIs work, what data is returned and how to structure the code around the API integration. The rapid prototyping from interactive development really boosts productivity. This post addresses the issue of controlling the output formatting in the interactive console. Not all types render as nicely as one would like.

### F# interactive

[fsi.exe](https://docs.microsoft.com/en-us/dotnet/fsharp/tutorials/fsharp-interactive/){:target="_blank"} is a tool for interactive development in F#. It comes with Visual Studio when F# tools are installed and is available in [VSCode](https://code.visualstudio.com/) with the [Ionide](http://ionide.io/) plugin. In an `*.fs` or `*.fsx` file simply highlight some code and hit `Alt + Enter` and the code will be evaluated and any output rendered in the interactive console window.

The output is usually formatted quite nicely, especially for F# record types. Here the output is basically an echo of the value declaration:

```fsharp
type Person =
  { FirstName : string
    LastName  : string }

let santa =
  { FirstName = "Santa"
    LastName  = "Claus" }
>
val santa : Person = { FirstName = "Santa"
                       LastName  = "Claus" }
```

For many non-`F#`-native types the output is not quite as readable. Take `DateTime` as an example:

```fsharp
> DateTime.Now;;
val it : DateTime = 2019-12-01 11.45.03 {Date = 2019-12-01 00.00.00;
                                         Day = 1;
                                         DayOfWeek = Sunday;
                                         DayOfYear = 335;
                                         Hour = 11;
                                         Kind = Local;
                                         Millisecond = 466;
                                         Minute = 45;
                                         Month = 12;
                                         Second = 3;
                                         Ticks = 637107975034665950L;
                                         TimeOfDay = 11:45:03.4665950;
                                         Year = 2019;}
```
This is a trivial example, but say you had a record type with one or more `DateTime` properties, the output would quickly drown in the verbosity of the `DateTime` properties. Other types with similar formatting include `Uri`, `TimeSpan` and `DateTimeOffset`. Enums do not look too great either by default:
```fsharp
> DayOfWeek.Monday;;
val it : DayOfWeek = Monday {value__ = 1;}
```
Luckily there is a way that we can take control of how to format any type just the way we want.

### Print transformers
In `*.fsx` files there is a globally available value named `fsi` with a few methods and values that can be used to modify the behavior of the current interactive session. Here I will discuss the method `fsi.AddPrinter<'a>('a -> string)`.

It is a generic function that takes a callback that takes a value of type `'a` and returns the string to be printed to the console.

Here is an how to set an improved `DateTime` format:
```fsharp
fsi.AddPrinter<DateTime>(fun dt -> dt.ToString("s"))
```
Whenever a `DateTime` value is printed to the interactive console it will be nicely formatted.
```fsharp
> DateTime.Now;;
val it : DateTime = 2019-12-02T19:55:51
```

In addition any `DateTime` properties of existing types will also use this formatting. Our `person` from before now renders like this:

> TODO Fix type

```fsharp
> person;;
val it : Person = { FirstName = "Santa"
                    LastName = "Claus"
                    DateOfBirth = 1980-06-12T12:03:33 }
```
For a little more elaborate formatting, the following function can be used:
```fsharp
fsi.AddPrinter<DateTime>(fun dt ->
  let time = dt.TimeOfDay
  let format =
    if   time.Ticks        = 0L then "yyyy-MM-dd"              // No time component
    elif time.Seconds      = 0  then "yyyy-MM-dd HH:mm"        // No seconds
    elif time.Milliseconds = 0  then "yyyy-MM-dd HH:mm:ss"     // No milliseconds
                                else "yyyy-MM-dd HH:mm:ss.fff"
  let str = dt.ToString(format, CultureInfo.InvariantCulture)
  sprintf "%s (%O)" str dt.Kind)
```
This formatting has two benefits. First, the format is as short as possible. If the `DateTime` value has no time component, only the date is shown. Likewise, second and millisecond components are only printed if they are non-zero. Second, the `DateTimeKind` property is explicitly printed. I find this very useful when debugging code that uses `DateTime` values, the `Kind` property is notoriously cumbersome to work with (link). 

```fsharp
> DateTime.Now;;
val it : DateTime = 2019-12-02 20:00:18.766 (Local)
```
where date only values are nice and short:
```fsharp
> DateTime.Today;;
val it : DateTime = 2019-12-02 (Local)
```

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
It would be tedious to copy this code in to every new `*.fsx` file. I like to put these customizations into a central file called `prelude.fsx` ([from Haskell](https://wiki.haskell.org/Prelude))

In order to load the printers in every session the `--use` directive can be specified when starting `fsi.exe`.

> In older versions of `fsi.exe` you might have to use `--load` instead of `--use`.

In Visual Studio go to `Tools > Options > F# Tools` and pass extra parameters like this:
![Visual Studio Prelude](/assets/path_to_prelude_visual_studio.png "Visual Studio Prelude")

In Visual Studio Code, go to `File > Preferences > Settings` or hit `Ctrl+,`, search for `Fsi Extra Parameters`, click `Edit in settings.json`:

![Ionide settings](/assets/ionide_settings.png "Visual Studio Prelude")

and add the following somewhere in the file:
```json
"FSharp.fsiExtraParameters": [
    "--use:C:\\path\\to\\prelude.fsx"
]
```
Now any new `fsi` session will have pretty printed types on by default.

<!-- ### Final bits
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
``` -->

### Additional reading
[F# Interactive Tips and Tricks: Formatting data](https://blogs.msdn.microsoft.com/dsyme/2010/01/08/f-interactive-tips-and-tricks-formatting-data-using-addprinter-addprinttransformer-and-a-in-sprintfprintffprintf/){:target="_blank"}

[F# Interactive Tips and Tricks: Visualizing data](https://blogs.msdn.microsoft.com/dsyme/2010/01/08/f-interactive-tips-and-tricks-visualizing-data-in-a-grid/){:target="_blank"}
