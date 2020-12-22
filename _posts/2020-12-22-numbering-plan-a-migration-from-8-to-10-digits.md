---
title: Numbering plan: A migration from 8 to 10 digits
description: Mobile phone: a tool that changed the way....
layout: post
author: Romuald Ouattara
excerpt_separator: <!--more-->
categories: [Android]
tags: [Android, Java, Kotlin, Numbering]
---

Mobile phone: a tool that changed the way we are communicating today. Today more and more people have access to mobile phones whether it's smart or not.
And what's the main goal of mobile phones ? Calling. Calling requires you to have a [subscriber identification module](https://en.wikipedia.org/wiki/SIM_card) aka **SIM**.
That SIM card is generally (if not always, in case of telephony at least) associated with a phone number.


<!--more-->

## Phone number

A phone number is sequence of a given number of digits. Those digits put together must be unique in your country. Your phone number is an ID that allows your operator to identify you.
To make sure all numbers are unique each country has it's own regulator. In Côte d'Ivoire the regulator is the [ARTCI](https://www.artci.ci/). The regulator determines how many digits should each
phone number have. That number must be big enough so that each citizen can have at least one phone number. In Côte d'Ivoire the number of digits for each number at the time of writing is 8. How many phone numbers of 8 digits can we generate ? Do the math :). Each phone number has a format:

```bash
# General phone number (8 digits)
XX-XX-XX-XX # where X is a digit between 0 and 9.

```

The truth about those 8 digits is that the first two digits belong to an operator (Moov, MTN, etc.). So we can generalize like this:

```bash
# General phone number (8 digits)
AB-XX-XX-XX # where A, B and X are a digits between 0 and 9. AB belong to an operator.

```

That gives an operator 6 digits to generate all the possible phone numbers for it's customers (at this point you should make sure you do the math correctly ;)). To simplify let's say we have two operators:

- **Operator 1** whose customers numbers will start by **99**  
- and **Operator 2** whose customers numbers will start by **11**

The possible phone numbers now are:

```bash
# General phone number (8 digits)
99-XX-XX-XX # where X is a digit between 0 and 9.
11-XX-XX-XX # where X is a digit between 0 and 9.
```

We suddenly have less possibilities to generate phone numbers. Once we don't have enough room to generate new phone numbers what can we do ? We increase the number of digits. But we have to be smart about how we increase that.
Some well known solutions consists in adding digits:

- at the beginning of the old phone number
- or at the end of the old phone number

If we decide to migrate all phone numbers from 8 to 10 digits the phone number format will be:
```bash
# case 1:
MP-99-XX-XX-XX # where X is a digit between 0 and 9.
MP-11-XX-XX-XX # where X is a digit between 0 and 9.

# case 2:
99-XX-XX-XX-MP # where X is a digit between 0 and 9.
11-XX-XX-XX-MP # where X is a digit between 0 and 9.
```

This means that each citizen will have to add MP (start or end) to all phone numbers in it's contacts. The same goes for developers who saved 8 digits phone numbers into their database.
To save you this complexity I introduce [numbering plan](https://github.com/ouattararomuald/numbering-plan). A library that will help developers migrate phone numbers from **old format** to the **new format**.


## Numbering plan

In this article I took Côte d'Ivoire as an example to simplify things. But `numbering plan` is not specific to Côte d'Ivoire. Instead it's a configurable library.

### Usage

First you start by defining the migration rules for you country

```kotlin
val ivoryCoastPlanFactory: CountryPlan = CountryPlan.Builder()
  .setOldPhoneNumberSize(8)
  .setInternationalCallingCode("225")
  .setDigitMapperPosition(Position.START) // Position.END
  .setMigrationType(MigrationType.PREFIX) // MigrationType.POSTFIX
  .setPrefixesMapper(
    mapOf(
      "07" to "07", // Orange
      "08" to "07", // eg: 08 XX XX XX => 07 08 XX XX XX (if MigrationType.PREFIX is used) => 08 XX XX XX 07 (if MigrationType.POSTFIX is used)
      "09" to "07",
      "04" to "05", // MTN
      "05" to "05",
      "06" to "05",
      "01" to "01", // MOOV
      "02" to "01",
      "03" to "01"
    )
  ).build()
```

Let’s break down all that:

- `setOldPhoneNumberSize(8)`: Means that your country has 8 digits phone numbers before the migration.
- `setInternationalCallingCode("225")`: The International Calling Code for you country (USA: 1, France: 33, UK: 44, etc.).
- `setMigrationType(MigrationType.PREFIX)`: Explained below.
- `setPrefixesMapper(...)`: Key-Value pairs of digits. They key is the old digits and the value is new digits to add to existing phone number.
To understand let's assume that your phone number is `08 XX XX XX` and your regulator deciced that operators must add `07` to all phone numbers starting with `08`. The regulator decides where they must add the new .PREFIX".
If they decide to add the new.PREFIX at the beginng of the phone number then your new phone number will be `07 08 XX XX XX` at the end of the migration. As a general library we must think about different use cases. That's why we introduced `MigrationType.PREFIX|MigrationType.POSTFIX` which determines where to add the new.PREFIX. If it must be added before the old phone number then use `MigrationType.PREFIX`. Otherwise you should use `MigrationType.POSTFIX` that will add the new digits after the old phone number.
- `setDigitMapperPosition(Position.START)`: In the example above we had to add (start or end) `07` to phone numbers starting by `08` (this corresponds to `Position.START`). But what if you want to add (start or end) `07` to old phone numbers ending with `08` ? For the last use case, you should use `Position.END`.

Then you pass those rules to `NumberingPlan` and you can start the migration:

```kotlin
val ivoryCoastPlanFactory: CountryPlan = ...
val numberingPlan = NumberingPlan(ivoryCoastPlanFactory)
val newPhoneNumbers = numberingPlan.migrate(
  mapOf(
    "userId-1" to " 00 22503 060 701 ",
    "userId-2" to " 00 225-03-060-701"
  )
)
```

After the migration `newPhoneNumbers` will be equal to:

```kotlin
mapOf(
  "userId-1" to "002250806070907",
  "userId-2" to "002250606070905",
  "userId-3" to "002250306070101",
  "userId-4" to "002250306070101",
  "userId-5" to "002250306070101"
)
```

Invalid phone numbers are removed. `migrate` is a **synchronous** method that takes as input the list of phone numbers to migrate and returns the new phone numbers. Only valid phone numbers will migrate.
A valid phone number is a phone number whose size is the one specified by `setOldPhoneNumberSize` and that of course contains only digits, spaces and/or dashes.


### Limitations

At the time of writing, here are some known limitations:

- Migrates only valid phone numbers.
- Can add new digits **before** or **after** the old phone number. We decided to not handle **in between insertions**.
- Can only look for digits to replace at the start or the end of old phone numbers. We think this shouldn't be a problem (generally) but as of now we decided to not handle such cases.
- It is synchronous. This is a choice that will let you pick the any library you want to handle async tasks.

The limitations may (will) change. That's why you should always check the project repo at [https://github.com/ouattararomuald/numbering-plan](https://github.com/ouattararomuald/numbering-plan). The library is open source and of course **contributions** are welcome.
