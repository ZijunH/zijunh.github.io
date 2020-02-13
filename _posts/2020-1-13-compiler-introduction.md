---
layout: post
title: "Compiler in Rust - Introduction"
date: 2020-01-13 15:12:54.000000000 +8:00
excerpt: Journey of writing a complete compiler in Rust without external infrastructure.
tags: 
  - discussion
  - compiler
keywords: [compiler, rust, project]
category: compiler
---

## Introduction

Once a professor told me every university student need to do three things before they graduate: write a compiler, write an OS, write a GPU rendering engine. So I decided to start with the compiler, since it appears to be the easiest. Since I have no experience in software projects, Rust, or compilers pipelines, I want to do all 3 at the same time. It seems pretty easy, what can go wrong?

## Goal

The goal I have in my head is a full compiler without any additional infrastructure. This mean no LLVM, no flex, no Yacc and I need to write a program that completes the process from human-readable code to machine code. This will mainly be a learning excercise, so don't expect this to be bug-free or fully featured. For each component, I hope it will be as generic as possible, which means I should be able to change a lot of things through a simple config file. Speed will not be a high priority; however, I will aim to implement the most efficient algorithms (or perhaps the most complex ones since this is an excercise) that I can find.

After this project is completed I hope the final product has the following stages:

- Front end 
  - A lexer that tokenises the text
  - A parser that parses the tokens into rules
  - Intermediate code generator that generates the rules into intermediate code
- Back end
  - Code optimiser that optimises the intermediate code
  - The actual compiler that converts intermediate code into assembly
  - Assembler that converts assembly into machine code

In order to complete this, I will be using the book "[Compilers: Principles, Techniques, and Tools](https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools)". From my understanding, this book is quite theoretical. Though this may increase the development time and probability of bugs, I believe it will make this project a great learning excercise.

The langauge that I will be compiling is [CMU's C0](http://c0.typesafety.net/tutorial/). Its full specifications can be found [here](http://c0.typesafety.net/doc/c0-reference.pdf). This was chosen because it is a subset of C, which means in the future I can expand this project into a compiler for C. Furthermore, it is relatively small and simple, which should allow me to get a prototype out pretty fast.

The time I expect this will take is around 6 months of coding. I have already spent 2 months reading through the compilers book, which provided me with a lot of theory and an idea of what to do. As this time period also coincides with the start of the 1st semester of my final year of undergrad, I may or may not finish within my planned timeframe.

I plan to record my journey throughout this project. As this is written in Rust, which I am very unfamiliar with, I will probably complain a lot about it as well.

Viel Glück!
