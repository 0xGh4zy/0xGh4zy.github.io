---
title: "OS Command Injection Skills Assessment (Arabic)"
date: 2026-05-03 00:00:00 +0200
categories: [HTB , Web Penetration Tester Path]
tags: [os-injection, HTB,CWES , Arabic]
description: "Exploiting OS command injection in a file manager application by bypassing blacklisted operators, commands, spaces, and slashes via environment variables."
image:
  path: /assets/img/posts/file-manager/cover.png   

---

<div dir="rtl" markdown="1">

بسم الله الرحمن الرحيم

اللهم علمنا ما ينفعنا و انفعنا بما علمتنا و زدنا بك علما

</div>

---

## Overview

<div dir="rtl" markdown="1">

الchallenge عبارة عن موقع `file manager`، الهدف منه استغلال `OS command injection` في الـ `parameters` بتاعة عمليات الـ `file system` عن طريق:

1. تحديد الـ `parameters` المحتمل يكون فيها `injection`
2. عمل `bypass` للـ `blacklisted operators`
3. عمل `bypass` للـ `blacklisted commands`
4. عمل `bypass` لـ `space` و `/` عن طريق `environment variables` عشان نقرء محتوي الـ `flag.txt`

</div>

---

## Recon

<div dir="rtl" markdown="1">

 بعد ما عملت `login` علي الموقع بالـ `guest user` لقيت قدامي البيدج دي:

</div>

![Home page](/assets/img/posts/file-manager/01-home.png)

<div dir="rtl" markdown="1">

دخلت عليهم واحد واحد و في نفس الوقت براقب الـ `url` فوق و اشوف الـ `parameters` اللي بتظهر و ايه المكان اللي ممكن يكون مستخدم فيها `OS Command`.

فلما دخلت علي `tmp folder` لقيته فاضي بس اتحط اسمه فوق في `to` parameter:

</div>

```
http://154.57.164.81:30115/index.php?to=tmp
```

![tmp folder](/assets/img/posts/file-manager/02-tmp-folder.png)

<div dir="rtl" markdown="1">

فده ممكن يدل إن قيمة `to` بتستخدم لتحديد الـ `directory target`، وممكن تكون داخلة في `file system operation`.

رجعت تاني للـ `home` و دوست ع اول فايل لقيت `parameter` جديد ظهر اسمه `view` بيتحط فيه اسم الفايل:

</div>

![view parameter](/assets/img/posts/file-manager/03-view-param.png)

<div dir="rtl" markdown="1">

رجعت تالت للـ `home`، جنب كل `file` في بعض الـ `buttons` كل واحد بيعمل حاجة:

- **الأول:** بيخليك تشوف المحتوي بتاع الملف عادي و بيستخدم `view`
- **الثاني:** لما دوست عليه فتحلي بيدج جديدة و ظهر `from` parameter محطوط فيه اسم الفايل، و الـ page دي فيها زرارين:
  - واحد بياخد الملف `copy` لـ `folder` تاني
  - والتاني بينقل الملف للـ `folder` اللي هختاره

</div>

![file buttons](/assets/img/posts/file-manager/04-file-buttons.png)

<div dir="rtl" markdown="1">

اخترت `temp folder` اني هنقل فيه، هنا بقا ممكن يكون `mv` مستخدم:

</div>

![move to tmp](/assets/img/posts/file-manager/05-move-tmp.png)

<div dir="rtl" markdown="1">

بعد منقلته الـ `URL` بقي كده:

</div>

```
http://154.57.164.81:30115/index.php?to=tmp&from=51459716.txt&finish=1&move=1
```

![after move](/assets/img/posts/file-manager/06-after-move.png)

<div dir="rtl" markdown="1">

بكده نكون شوفنا الاماكن اللي محتمل يبقي فيها `OS command` مستخدم، نبدأ نتست الاماكن دي بقا.

اول مكان هتست فيه هو المكان اللي بننقل من خلاله file من مكان لمكان، ليه؟
 عشان جربت باقي الاماكن لقيت ان ده الـ `vulnerable` 😄 فاختصارا لطول الرايتب يلا بينا.

</div>

---

## Understanding the Injection Points

<div dir="rtl" markdown="1">

قبل منتست عايزين نفهم حاجة بسيطة، المفروض في الـ `request` اللي بيعمل `move` للـ `file` من مكان لمكان في اتنين `parameters` اللي هما `to` و `from`:
</div>

- `to` : contains destination folder — ده المكان اللي هيروحله
- `from` : contains source path — مكان الفايل يعني
<div dir="rtl" markdown="1">
امر `mv` بيبقي كده:

</div>

```bash
mv source_path destination_path
```

<div dir="rtl" markdown="1">

يعني لو في المثال بتاعنا:
</div>
- `source path` = `/var/www/html/files/51459716.txt`
- `destination path` = `/var/www/html/files/tmp`
<div dir="rtl" markdown="1">
فالامر هيبقي كده:

</div>

```bash
mv /var/www/html/files/51459716.txt /var/www/html/files/tmp
```

<div dir="rtl" markdown="1">

فلو هتعمل `injection` في الـ `to` parameter هيبقي تمام، لكن لو عملت `injection` في الـ `from` parameter هتحتاج تعمل `comment` بـ `#` بعد الامر اللي هتكتبه عشان يتجاهل الـ `destination path`. دي نقطة كنت حابب اوضحها الاول.

</div>

---

## Bypassing the Blacklisted Operators

<div dir="rtl" markdown="1">

اول حاجة هنجربها هي الـ `operator` المناسب من دول:

</div>

| **Injection Operator** | **Injection Character** | **URL-Encoded Character** |
| --- | --- | --- |
| Semicolon | `;` | `%3b` |
| New Line | `\n` | `%0a` |
| Background | `&` | `%26` |
| Pipe | `\|` | `%7c` |
| AND | `&&` | `%26%26` |
| OR | `\|\|` | `%7c%7c` |
| Sub-Shell | ` `` ` | `%60%60` |
| Sub-Shell | `$()` | `%24%28%29` |

<div dir="rtl" markdown="1">

جربت اعمل `injection` في `to` parameter الاول، طبعا هنحتاج `operator` معموله `url encode` فجربت احط `3b%` بعد الـ `tmp` اللي هو الـ `destination path`، المفروض الامر هيبقي كده:

</div>

```bash
mv /var/www/html/files/51459716.txt /var/www/html/files/tmp;
```
<div dir="rtl" markdown="1">
جابلي `error` malicious request denied
</div>
![semicolon blocked](/assets/img/posts/file-manager/07-semicolon-blocked.png)

<div dir="rtl" markdown="1">

 كده واضح إن `;` غالبًا معمول له `blacklist`. بعدها جربت `%0a` نفس الكلام، بعدها جربت `%26` اشتغلت تمام 

</div>

![ampersand works](/assets/img/posts/file-manager/08-ampersand-works.png)

<div dir="rtl" markdown="1">

بكده عملنا `bypass` للـ `blacklisted operators`.

</div>

---

## Bypassing the Command Blacklist

<div dir="rtl" markdown="1">

نجرب بقا الـ `commands`، هنبدأ نجرب `whoami` جابلي `malicious request denied`:

</div>

![whoami blocked](/assets/img/posts/file-manager/09-whoami-blocked.png)

<div dir="rtl" markdown="1">

جربت `ls` برضو نفس الكلام، يبقي كده احتمال يكون في `blacklist`. نجرب نعمل `bypass`.

اول حاجة هجرب أضيف `single quote` `'` لأنها بتكسر الـ `blacklist pattern`، لكن الـ `shell` بيعيد تجميعها ويفهم الأمر بشكل طبيعي، بس لازم يبقو عددهم زوجي:

</div>

```
%26w'h'oami
```

![quotes bypass](/assets/img/posts/file-manager/10-quotes-bypass.png)

<div dir="rtl" markdown="1">

اشتغلت و الامر اتنفذ فعلا ، كده عملنا `bypass` للـ `operator` والـ `command`.

</div>

---

## Getting the Flag

<div dir="rtl" markdown="1">

مطلوب مننا نجيب الـ `flag` اللي موجود في `flag.txt/`، المفروض نكتب:

</div>

```
%26c'a't+/flag.txt
```

![space slash blocked](/assets/img/posts/file-manager/11-space-slash-blocked.png)

<div dir="rtl" markdown="1">

يبقي كده الـ `space` و `/` شكلهم معمول لهم `blacklist` هما كمان، محتاجين نعملهم `bypass`.

نجرب ناخد قيمتهم من `environment variables`:
</div>
- `/` = `${PATH:0:1}`
- `space` = `${IFS}`
<div dir="rtl" markdown="1">
يبقي الـ `payload` هيبقي شكله كده بعد اضافتهم:

</div>

```
%26c'a't${IFS}${PATH:0:1}flag.txt
```

![flag](/assets/img/posts/file-manager/12-flag.png)

<div dir="rtl" markdown="1">

بكده تكون انتهت المهمة بنجاح ، ضن✅

</div>

---

## Bonus — Injection in from Parameter

<div dir="rtl" markdown="1">

طيب ماذا لو كنا هنعمل `inject` في الـ `from` parameter؟

زي مقولتلك هنحتاج في الاخر نعمل `comment`، عشان الامر يبقي كده:

</div>

```bash
mv /var/www/html/files/51459716.txt&whoami&#   /var/www/html/files/tmp
```

<div dir="rtl" markdown="1">

فكده هنحتاج نضيف `#` بعد الامر اللي هنكتبه:

- `#` = `%23` (url encode)

يبقي الـ `payload` بتاعنا هيبقي:

</div>

```
%26w'ho'ami%26%23
```

![from injection whoami](/assets/img/posts/file-manager/13-from-whoami.png)

<div dir="rtl" markdown="1">

> **Note:** مع `cat` مش هنحتاج نعمل `comment`، ليه؟
>
> `cat` عادي تحط بعدها كذا `path` لانها هتعرضلك محتوي الـ `file` لكل `path`، بس هي هتدي `error` بعد متعرض محتوي الـ `flag.txt` لان `tmp` ده مش `file` لكن `directory`، بس كده .بس لو مش عايزين الـ `error` يظهر هنعمل `comment`.
{: .prompt-info }

</div>

```bash
mv /var/www/html/files/51459716.txt&cat /flag.txt /var/www/html/files/tmp
```

![from injection cat](/assets/img/posts/file-manager/14-from-cat.png)

![flag from](/assets/img/posts/file-manager/15-flag-from.png)

---

<div dir="rtl" markdown="1">

 و بكده تكون وصلت لنهاية الرايتب يا عزيزي القارئ ، شكرا لقرأتك لحد هنا و اتمني تكون استفدت ان شاء الله و لا تنسونا من صالح دعائكم.

وطبعا لو محتاج تسأل علي اي حاجة او في تعليق علي حاجة تقدر تبعتلي في اي وقت:

- **GitHub:** [github.com/0xgh4zy](https://github.com/0xgh4zy)
- **LinkedIn:** [linkedin.com/in/youssef-wael-mohamed](https://linkedin.com/in/youssef-wael-mohamed)
- **Discord:** `0xgh4zy`

> «إن أحسنت فمن الله، وإن أسأت أو أخطأت فمن نفسي والشيطان»

</div>