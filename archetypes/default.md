+++
title = '{{ replace .Name "-" " " | title }}'
date = {{ .Date }}
draft = true

categories = ['Dev']
tags = ['hugo', 'papermod']
+++

# Nagłówek H1

Jakis tekst z **pogrubieniem** i *kursywą*.

- Punkt listy 1
- Punkt listy 2

1. Krok pierwszy
2. Krok drugi

```ts {linenos=true hl_lines="2-3"}
const a = 1;
const b = 2;
console.log(a + b);
