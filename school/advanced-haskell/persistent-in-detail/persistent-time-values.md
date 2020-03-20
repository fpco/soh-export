# Time values in persistent

# Overview

Time values can be stored in databases in a confusing variety of formats. These get mapped to a smaller set of Haskell types when you use the `persistent` database interface.  This is a short tutorial to look at the various possibilities, and recommend some best practices.

# Database types.
Besides the usual `DATE` and `TIME` types, which represent part of a complete time value, the usual types are `TIMESTAMP` and `TIMESTAMP WITH TIMEZONE`. `DATE` and `TIME` are handled quite well by the <hoogle search="Data.Time.Calendar.Day">Day</hoogle> and <hoogle search="Data.Time.LocalTime.TimeOfDay">TimeOfDay</hoogle> types.

Complete - or near complete - time values like `TIMESTAMP` and `TIMESTAMP WITH TIMEZONE` are a bit more complicated. 

## `TIMESTAMP` without a timezone
The normal mapping for a `TIMESTAMP` is <hoogle search="Data.Time.Clock.UTCTime">UTCTime</hoogle>. This is a timestamp with an implicit timezone of UTC. There is no implicit timezone on the SQL side. If anything, it's assumed to be local time. However, that information isn't available in the database. This mapping is used because it provides a proper two-way mapping, but the semantics on the Haskell side may be wrong. Best practices will prevent that.

## `TIMESTAMP WITH TIMEZONE`
These are mapped to <hoogle search="Data.Time.LocalTime.ZonedTime">ZonedTime</hoogle>. This is the preferred map if you need to deal with time zones. However, the PostGRESql database actually stores values in UTC and translates to the local time zone on output. The local time zone setting in the library is UTCTime, so these give you the same effect as using UTCTime, and will be translated to that in a later version of the library.

# Best Practices
As with any complex data structure, the best practice for storing time is to convert the values to a canonical form on input, and convert back if needed for display.

## `TIMESTAMP WITH TIMEZONE`
If the time zones that go with the recorded time are important, then the only choice is to use `TIMESTAMP WITH TIMEZONE` and `ZonedTime`.  Comparing values and calculating intervals requires conversion to a command - and preferably monotonic - timezone.

## `TIMESTAMP` without a timezone
If you just need to know the order of events that are being timestamped, or intervals between them, the converting all values to UTC and using `TIMESTAMP` and `UTCTime` works very well.

# A note on intervals
None of the time types here deal properly with [leap seconds](http://en.wikipedia.org/wiki/Leap_seconds). If you are dealing with long intervals that need more than one or two second accuracy, these types may not be appropriate for your use.
