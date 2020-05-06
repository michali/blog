---
layout: post
title:  "NUnit Test method code snippet"
date:   2013-11-28 00:00
categories:
- Automated Tests
- Visual Studio
---

This is if you don't use Resharper. Suffices to say Resharper's Live templates are much easier to create and update but Visual Studio code snippets are an alternative option.

```xml
<?xml version="1.0" encoding="utf-8"?>
<CodeSnippets xmlns="http://schemas.microsoft.com/VisualStudio/2008/CodeSnippet">
  <CodeSnippet Format="1.0.0">
    <Header>
      <Title>
        NUnit Test Method Insertion
      </Title>
      <Description>Inserts a template for an automated test</Description>
      <Shortcut>test</Shortcut>
    </Header>
    <Snippet>
      <Declarations>
        <Literal>
          <ID>UOW</ID>
          <ToolTip>The unit of work under test</ToolTip>
          <Default>UnitOfWork</Default>
        </Literal>
        <Literal>
          <ID>ACT</ID>
          <ToolTip>The stimulant to apply to the system under test</ToolTip>
          <Default>Action</Default>
        </Literal>
        <Literal>
          <ID>EB</ID>
          <ToolTip>The expected outcome after the stimulant has been applied</ToolTip>
          <Default>ExpectedBehaviour</Default>
        </Literal>
      </Declarations>
      <Code Language="CSharp">
        <![CDATA[[Test]
        public void $UOW$_When$ACT$_$EB$()
        {
          $end$
        }]]>
      </Code>
    </Snippet>
  </CodeSnippet>
</CodeSnippets>
```

You can change this according to your wants/needs but this snippet will produce the following once you call it with the <code>test</code> shortcut:

```csharp
[Test]
public void UnitOfWork_WhenAction_ExpectedBehaviour()
{
   //cursor will terminate here
}
```

UnitOfWork, Action and ExpectedBehaviour are the default text that replaces three tokens in the snippet body. The three tokens are bordered by dollar signs and will become highlighted and editable once the snippet is inserted.

A code snippet for xUnit is similar, with a Fact attribute instead of a Test one.

So you define the placeholder tokens in the Declarations part of the snippet and again you use them in the snippet body. Resharper has condensed this into just a snippet body with just the default text.

Many thanks to [Roy Osherove](https://osherove.com) for the Resharper test Template idea which he's mentioned in some of his videos.

**Edit:** This only took eight months. Removed some newlines in the snippet and added an access modifier.