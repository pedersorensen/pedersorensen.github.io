---
layout: post
excerpt: Controlling f# interactive output
categories: programming fsharp fsi interactive
---

This post is an entry in the [2019 F# advent calendar](https://sergeytihon.com/2019/11/05/f-advent-calendar-in-english-2019/){:target="_blank"} hosted by Sergey Tihon.

### Intro

A great deal of my development concerns integrating with internal and 3rd party APIs. To that end I find the interactive development provided by `F#` and `fsi` ([F# Interactive](https://docs.microsoft.com/en-us/dotnet/fsharp/tutorials/fsharp-interactive/){:target="_blank"}) extremely helpful when figuring out how the APIs work, what data is returned and how to structure the code around the API integration. This post describes how to control the output formatting in the F# interactive console.

### F# interactive

[fsi.exe](https://docs.microsoft.com/en-us/dotnet/fsharp/tutorials/fsharp-interactive/){:target="_blank"} is a tool for interactive development in F#. It comes with Visual Studio when F# tools are installed and is available in [VSCode](https://code.visualstudio.com/){:target="_blank"} with the [Ionide](http://ionide.io/){:target="_blank"} plugin. In an `*.fs` or `*.fsx` file simply highlight some code and hit `Alt + Enter` and the code will be evaluated and the result rendered in the interactive console.

The output is usually formatted quite nicely, especially for F# record types, as this example shows:

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

For most regular .NET types the formatting is not always this good. Take `DateTime`'s and `Enum`'s

```fsharp
type DateAndDay =
  { Date : DateTime
    Day  : DayOfWeek }

let christmasDay2019 =
  { Date = DateTime(2019, 12, 24)
    Day  = DayOfWeek.Tuesday }

> christmasDay2019;;
val it : DateAndDay =
  { Date = 2019-12-24 00.00.00 {Date = 2019-12-24 00.00.00;
                                Day = 24;
                                DayOfWeek = Tuesday;
                                DayOfYear = 358;
                                Hour = 0;
                                Kind = Unspecified;
                                Millisecond = 0;
                                Minute = 0;
                                Month = 12;
                                Second = 0;
                                Ticks = 637127424000000000L;
                                TimeOfDay = 00:00:00;
                                Year = 2019;}
    Day = Tuesday {value__ = 2;} }
```

> And yes, in Denmark we celebrate Christmas on the 24th of December!

The `Date` rendering is far to verbose and the weird `value__` in the `Day` enum smells a bit like an internal value, irrelevant for most purposes. Other types with similar formatting include `Uri`, `TimeSpan` and `DateTimeOffset` and when working with these types the interactive output can easily become drowned in verbose formatting.

Luckily there is a way that we can take control of how to format any type just the way we want.

### Output printers
In `*.fsx` files there is a globally available value called `fsi` with a few methods and values that can be used to modify the behavior of the current interactive session. Here I will discuss the method `fsi.AddPrinter<'a>('a -> string)`.

`AddPrinter` is a generic function that takes a formatter function for a given type `'a` that must return the string to be printed to the console.

Here is an how to set a custom `DateTime` format:
```fsharp
fsi.AddPrinter<DateTime>(fun dt -> dt.ToString("s"))
```
`DateTime` values printed to the interactive console will now be nicely formatted:
```fsharp
> DateTime.Now;;
val it : DateTime = 2019-12-05T11:10:23
```
The `Enum` formatting can be improved using this printer, which in addition to the value also prefixes the type name:
```fsharp
fsi.AddPrinter<Enum>(fun x -> sprintf "%s.%O" (x.GetType().Name) x)
```
`DateTime` and `Enum` properties of existing types will now also use this formatting. The `christmasDay2019` value now looks like this:
```fsharp
> christmasDay2019;;
val it : DateAndDay = { Date = 2019-12-24T00:00:00
                        Day = DayOfWeek.Tuesday }
```
For further improvement I have found the following `DateTime` formatter to be quite helpful:
```fsharp
fsi.AddPrinter<DateTime>(fun dt ->
  let time = dt.TimeOfDay
  let format =
    if   time.Ticks        = 0L then "yyyy-MM-dd"              // No time component
    elif time.Seconds      = 0  then "yyyy-MM-dd HH:mm"        // No seconds
    elif time.Milliseconds = 0  then "yyyy-MM-dd HH:mm:ss"     // No milliseconds
                                else "yyyy-MM-dd HH:mm:ss.fff"
  let str = dt.ToString(format, System.Globalization.CultureInfo.InvariantCulture)
  sprintf "%s (%O)" str dt.Kind)
```
This formatting has two benefits. First, the output is as short as possible. If the `DateTime` value has no time component, only the date is displayed. Likewise, second and millisecond components are only printed if they are non-zero. Second, the `DateTimeKind` property is explicitly printed. I find this very helpful when working with code that uses `DateTime`, the `Kind` property is [particularly difficult](https://blog.nodatime.org/2011/08/what-wrong-with-datetime-anyway.html){:target="_blank"} to work with. With this formatter the `Kind` is always visible.
```fsharp
> DateTime.UtcNow;;
val it : DateTime = 2019-12-05 14:50:18.467 (Utc)
> DateTime.Today;;
val it : DateTime = 2019-12-05 (Local)
```

The `christmasDay2019` value now looks like this:
```fsharp
> christmasDay2019;;
val it : DateAndDay = { Date = 2019-12-24 (Unspecified)
                        Day = DayOfWeek.Tuesday }
```
Hmm... Looks like there was a bug in my code from earlier, I forgot to set the `DateTimeKind` to `Local` when I declared the value earlier. That would have been harder to spot with the default formatting.

For my personal use I have added similar printers for `Uri`, `TimeSpan`, `DateTimeOffset`, `FileInfo` and `DirectoryInfo`.

This feature can, of course, also be used for domain specific types. If your domain types are complex it pays to create a custom printer such that the `fsi` output stays nice and readable. The [Deedle](http://bluemountaincapital.github.io/Deedle/){:target="_blank"} library uses similar techniques when rendering its data frame types.

### Auto loading printers
It would be tedious to copy this code into every `*.fsx` file. I like to put these customizations into a central file called `prelude.fsx` (hat tip to [Haskell](https://wiki.haskell.org/Prelude){:target="_blank"}). To load this file in every session an additional argument must be passed to `fsi.exe` upon startup.

In Visual Studio go to `Tools > Options > F# Tools` and pass extra arguments like this:
![Visual Studio Prelude](/assets/path_to_prelude_visual_studio.png "Adding extra argument to fsi in Visual Studio.")

In Visual Studio Code, go to `File > Preferences > Settings` or hit `Ctrl+,`, search for `Fsi Extra Parameters`, click `Edit in settings.json`:

![Ionide settings](/assets/ionide_settings.png "Adding extra argument to fsi in Ionide.")

and add the following somewhere in the file:
```json
"FSharp.fsiExtraParameters": [
    "--use:C:\\path\\to\\prelude.fsx"
]
```

> In older versions of `fsi.exe` you have to use `--load` instead of `--use`.

Now any new `fsi` session will have pretty printed types enabled by default.

### Additional reading
[F# Interactive Tips and Tricks: Formatting data](https://blogs.msdn.microsoft.com/dsyme/2010/01/08/f-interactive-tips-and-tricks-formatting-data-using-addprinter-addprinttransformer-and-a-in-sprintfprintffprintf/){:target="_blank"}

[F# Interactive Tips and Tricks: Visualizing data](https://blogs.msdn.microsoft.com/dsyme/2010/01/08/f-interactive-tips-and-tricks-visualizing-data-in-a-grid/){:target="_blank"}
