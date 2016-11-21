---
title: Time ticks on
excerpt: "Building a quant library always starts with the date!"
category: "tim"
header:
  overlay_image: timetickson/header.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: timetickson/teaser.jpg
tags: [rates, fixed income, ir, .net]
---

As described in previous posts by Gav we are going to tackle the idea of building rates curves first.
In order to do that the first thing we need is a number of date functions to allow us to generate
schedules to do day counts, year fractions and all the things needed to price a fixed income product.
So we have created a separate assembly project called Qwack.Dates.

![Calendar](/images/timetickson/calendar.jpg)

## Because we all need a holiday

The first step is to make a calendar class. It is not very complicated and is more just a container for the holiday/non-trading days for a specific
calendar. 

The important details are that we can blank out a specific day of the week always (so you don't have to 
include weekends, but can customize them for countries that have say Friday to Saturday weekends). We have included
months you can block out as well, we won't be using this right now but it will be useful when we get to futures that
don't have specific month maturities.

![Inherit](/images/timetickson/inherit.jpg)

## One inside another, inside another ....

The next feature is that calendars can inherit from other calendars. This is useful for common holidays such as easter
or other relationships (say a state calendar that needs to include all federal holidays in the US). We then have the 
ICalendarProvider interface. The idea behind the providers is that they will load calendars from a specific source and then
provide access directly to the raw calendars loaded from the source as well as a "CalendarCollection". This will provide 
pre-merged calendars for performance reasons, as well as being able to ask for multiple calendars to be merged, 
an example is below

``` csharp
var provider = CalendarsFromJson.Load(JsonCalendarPath);
```

or

``` csharp
var provider = CalendarsFromJson.Parse(yourStringOfJson);
var calendar = provider.Collection["nyc"];
var combinedCalendar = provider.Collection["nyc+lon"];
```

We have included a basic calendar file, useful for testing and a basic way to get started in /data/Calendars.json.
Now that we have calendars we can start to write the date functions themselves. To do that we need the concept of adding and 
subtracting "periods" from dates. The common one we all know is things like "1y" or "2bd" we have all seen it over and over.
To this end we have an enumeration that looks something like

``` csharp
public enum DatePeriodType
{
  BusinessDay = 0,
  B = 0,
  Day = 1,
  D = 1,
  Week = 2,
  W = 2,
  Month = 3,
  M = 3,
  Year = 4,
  Y = 4
}
```

![Frequency](/images/timetickson/frequency.jpg)

## All about the frequency

Then we have something called "Frequency" which contains a 
"number of periods" and a "period type"
this is used for most of the date functions to denote step sizes. This was made as a struct as it is just 2 int32 
(will fit in a single 64bit register). 

These are passed around a lot, and created and destroyed rapidly so a struct is 
the right choice here I feel. Because we need to do comparisons we had to provide an = operator, and therefore a !=, 
Equals(object otherObject), GetHashCode. The Equals and GetHashCode could probably be improved but we aren't really
using them yet, so we can revisit that at that time.

Initially I wrote a bunch of convinence static functions for commonly used periods such as 0bd, 1bd, 3month, 1year. 
However I realised that I needed more and more of these as I was writing further tests using the framework and it was 
going to become a mass of these functions. So instead, I added some extension methods on the int type to allow a more 
"fluent" interface for these. The extensions are on the int type, which is something I had to think
about as I am against needlessly adding extension methods on base classes that show up everywhere. In this case the fact 
that you have the dates namespace in your using will mean that you are interested in using dates, it's not in a "base" 
namespace so they will override and show up everywhere in your application. The Api now for using these is pretty neat

``` csharp
var myFrequency = 3.Months();
var myOtherFrequency = 2.Bd();
```

and so on. This makes writing big blocks of code (at this stage only tests but still) much clearer so instead of

``` csharp
Frequency[] swapTenors = { new Frequency(3, DatePeriodType.Y), new Frequency(4, DatePeriodType.Y), new Frequency(5, DatePeriodType.Y) };
```
We would now have something that looks like

``` csharp
Frequency[] swapTenors = { 3.Years(), 4.Years(), 5.Years() };  
```

The other method available is to use a string like

``` csharp
string[] swapTenors = { "18m", "2y", "3y", "4y", "5y", "7y", "10y", "15y", "20y" };
```
which while very succinct, requires string splitting and reading so isn't very efficient. But also has no compile time 
checking, which is less important in tests but one of the reasons you "pay the penalty" of less flexibility of a statically 
typed language is to get as much compile time checking as possible.
This method will of course need to be used when parsing/deserializing from text based formats later on.

![Standards](/images/timetickson/standard.jpg)

## Standards never are...

So now we have periods, frequencies and calendars we need to look at rolling "rules". Like all good "standards" there 
are plenty of rules. We have included

1. Following - move to the next one
2. Previous - move to the previous business day
3. Modified Following - Same as following unless it will go into a new month, in which case roll back
4. Modified Previous - Same as Modified Following but in reverse
5. Nearest Following - Sometimes confused with modified following, but rolls forward, if this is not a business day it will find the closest (forward or back) good day
6. Nearest Previous - Same as Nearest but attempts to roll previous first
7. LME Nearest - Specific to the LME roll rules see [LME Rulebook](https://www.lme.com/~/media/Files/Regulation/Rulebook/Part%204%20-%20Contract%20regulations.pdf)
8. Nearest Modified Following - A combination of Nearest Following and Modified Following (Also the same rule as the LME rules above)
9. Modified Following Libor - Modified Following adjusted for Libor specifics
10. IMM - International Money Markets rules used for specific instruments on the money markets.
11. EOM - End of month
12. ShortFLongMF - Short Following, Long Modified Following, uses two different roll types depending on if a period is a long or short stub period

So that about covers our needs for now, as always feel free to add an issue if there is one we have left off or you might 
find useful!

Now that is out of the way we need an enumeration of day counts. So we have come up with the following

1. Act/360
2. Act/365
3. Act/Act ISDA standards
4. Act/Act ICMA standards
5. Act/365 Fixed
6. 30/360 
7. Unity

To get a detailed overview of how these are calculated you can get started here 
[Day Counts](https://wiki.treasurers.org/wiki/Day_count_conventions). 

Having finished all of that (and thanks to the great 
work by [Gav](https://cetus.io/gav/) on all of the standards and actually implementing most of them!). I hope you
are getting a picture of why a standarised library is both important and useful. 

![Repeat](/images/timetickson/repeat.jpg)

## Leave repetition to nature

This date handling alone is done over and over on almost every project we have ever worked on, some libraries provide 
this (QuantLib is the obvious one) but they tend not to go into all of the details of the various different standards 
and leave that as an exercise for the reader, and math libraries understandably aren't in the business of the strange 
date functions as they aren't mathematical.

The rest of the code that is important is in the "DateExtensions" class. All of them have been written as static 
extension methods on the DateTime class. We are only using the date portion of the class and may consider a move to 
[nodatime](http://nodatime.org/) by the brilliant John Skeet further down the line but right now learning another way of 
doing date/time isn't high on my list.

We have a selection of the functions below

1. d.GetNextImmDate() - also previous available used for Internal Money Market Dates [IMM Dates](https://en.wikipedia.org/wiki/IMM_dates)
2. d.BusinessDaysInPeriod(endDate, calendar) - used to return a list of dates that are business dates between 2 inclusive dates
3. d.CalculateYearFraction(endDate, dayCountBasis) - returns a double with the fraction
4. d.[[First, Last]][[Business]]DayOfTheMonth - Simply returns the first or last business or calendar day of a specific month.
5. d.NthSpecificWeekDay(dayOfWeek) - Returns for instance the 2nd Monday of a month, there is also a helper that wraps it for the third Wednesday as it is often used
6. d.IfHolidayRoll(various) - Used to roll forward/back or based on the various rules above if the date supplied is not a good business day for the calendar

Finally we have AddPeriod and Subtract period, these are used by most of the functions above and are the most useful of 
the functions, it is the "heart" of the library.

![Cover](/images/timetickson/coverage.jpg)

## Some cover is always good

Last of all unit tests have been added, they certainly don't [cover](https://coveralls.io/builds/8790953) everything but 
they are a start and as it is a cornerstone of all the future calculations more should be added. However for now we have 
the basic functions we need to move on, I am sure we will be revisiting dates in the future having already raised three 
[tasks](https://github.com/cetusfinance/qwack/labels/DateFunctions) myself already.

Next up.. adding the various classes we actually need to be able to calculate prices on fixed income products, so one 
more post and then we get to the good stuff!   

P.S. Having had a look and trying to think about it, I am considering making a struct which is basically a datetime with an 
attached calendar... the reason for this is that then you could extend the fluent dates to something like

``` csharp
var dateWithCal = myDateTime.WithCalendar(usdCalendar);

var newDateWithCal = dateWithCal + 1.Bd();
```

Any thoughts?
