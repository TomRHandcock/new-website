---
title: "Lessons from building a Dart code generator"
summary: "An account of some of the learnings from implementing Flumepod."
date: 2026-08-29T12:00:00+01:00
series: ["posts"]
tags: ["Dart", "Flutter"]
showToc: false
TocOpen: false
draft: false
hidemeta: false
comments: false
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---
I have recently collaborated on [Flumepod](https://pub.dev/packages/flumepod), a code generator for bridging the gap between Riverpod and Mockito; allowing you to generate mocks of `Notifier` classes for stubbing and verification. This is a post about some of the things I learnt along the way that I wish I had known from the start, published in the hopes it will help other developers.

Code generators, especially those that use introspection on existing code, are an inherently messy domain. You are ultimately operating with untyped strings and even in a relatively streamlined language such as Dart, there are many edge cases you have to be mindful of, such as generic types, handling duplicate symbols when resolving imports, and how asynchronous code can impact [the "colour" of the output](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/). Not to mention keeping the generator source coherent and maintainable as the language specification changes overtime.

Most code generators rely on first analysing existing code in order to derive more code. For example, one of the most popular packages in the Dart ecosystem, [Freezed](https://pub.dev/packages/freezed), generates immutable "data classes" with a handful of utility methods from basic factory definitions. When building such a code generator, it may be tempting to try and abstract away all of the complexity in the Dart type system behind custom, simpler entity models that capture introspection details about the input code. For example, a custom `Type` class representing symbols in the input code which contains data points that are relevant to your use case. Certainly, this is what typical front-end development encourages, a separation of internal, application specific "entity" models from external, "domain" models to allow for weaker coupling between your program and other systems.

This is what early versions of Flumepod attempted to do, parse the raw `analyzer` models obtained from existing code analysis into internal entity models; neatly segmenting symbols the generator encountered (such as a notifier's `State` type) into categories that impacted the code needing to be generated with the use of a sealed class. This included cases such as custom user types which require import resolution, generic types which represent a tree of nested parameter types, and asynchronous types which require keywords in generated functions.

The problem is that creating this nice classification will ultimately be a leaky abstraction that will fail to represent all valid permutations of code that your generator might encounter. Overtime, you'll be forced to patch your custom type representation again and again until eventually you just end up with a Frankenstein of the [`DartType`](https://pub.dev/documentation/analyzer/latest/dart_element_type/DartType-class.html) and [`Element`](https://pub.dev/documentation/analyzer/latest/dart_element_element/Element-class.html) classes. It is primarily for this reason I have learned to disregard the aforementioned separation of models principle and opt to just operate on the raw `analyzer` types instead when introspecting on code.

Furthermore, operating on the types that Dart's `analyzer` package gives you forces you to stay up to date with changes in the language specification due to Dart's sound type system and exhaustiveness checks.

Type-safety brings me to my second recommendation which is to make use of a package that I didn't know about until later on, [`code_builder`](https://pub.dev/packages/code_builder). As the name implies, `code_builder` primarily provides type-safe builders for generating Dart code, instead of you having to manually manipulate strings with [`StringBuffer`](https://api.flutter.dev/flutter/dart-core/StringBuffer-class.html) or interpolation.

A simple generator for a class using the `code_builder` package might look as follows...

```dart
import 'package:code_builder/code_builder.dart';
import 'package:dart_style/dart_style.dart';

void main() {
	final clazz = Class((cb) => cb
		..name = "Cow"
		..methods.add(
			Method.returnsVoid((mb) => mb
				..name = "moo"
				..body = const Code("print('moo');")
			)
		)
	);
	final emitter = DartEmitter();
	final formatter = DartFormatter(
		languageVersion: DartFormatter.latestLanguageVersion
	);
	print(formatter.format("${clazz.accept(emitter)}"));
}
```

Which generates...
```dart
class Cow {
	void moo() {
		print('moo');
	}
}
```

Whilst this is a trivial example, the API improves the "glanceability" of the code to pick out which part you are interested in. Something you'll appreciate six months down the road when it comes time for maintenance.

Another very useful feature of the `code_builder` package is the top-level [`refer`](https://pub.dev/documentation/code_builder/latest/code_builder/refer.html) function. A naive code generator implementation may simply generate a series of import statements at the top of the output file for the types it determines it needs. However, this fails when two imported libraries both declare the same symbol, `MyClass` for example. To resolve this issue, most code generators either piggy-back off the input file via `part` directives or uniquely prefix imports. In the latter case, keeping track of what URI is prefixed with what can be non-trivial but `refer` handles this nicely by generating the import statements and matching the import prefixes automatically!

My final recommendation is to highlight the importance of dog-fooding the code generator on large, real code bases that multiple people contribute to. Whilst you can have all the unit and golden tests you want, there will be cases you won't anticipate. When you do find such a defect, be sure to add a test covering it in your automated testing to avoid it becoming a problem again in future.

One such case for Flumepod was when a colleague tried to generate a mock of a class that included a getter method, a pattern which I typically don't use when I write Riverpod notifiers, so unintentionally omitted from Flumepod. The mocked class simply didn't include a stub for the method so there was no means by which to stub or verify calls on the getter. In the end it was a simple fix to take note of getter and setter declarations in the original notifier class definitions but it took another pair of eyes to highlight the issue which I otherwise would have been blind to.

I hope this brief account of my learnings from implementing a code generator proved insightful and will help you to avoid falling into the same traps. Also, if you do routinely use Riverpod for state management in Dart/Flutter, perhaps give [Flumepod](https://pub.dev/packages/flumepod) a go to improve your unit testing workflow. Lastly, thank you for reading!