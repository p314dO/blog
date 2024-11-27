---
title: I Sommari
date: 2023-05-02
description: Setup table of content in Hugo blog awesome theme
---

## Sommario

Questo tema supporta la visualizzazione di un sommario (Table Of Content - ToC) all'interno degli Articulos.

## Abilitare il sommario in tutti gli Articulos

Per abilitare il sommario in tutti gli Articulos (globalmente) è necessario impostare il parametro `toc` a `true` in `hugo.toml`.

```toml
[params]
  toc = true
```

Per disabilitare il sommario globalmente invece, basta semplicemente cancellare il parametro `toc` oppure impostarlo a `false`.
 
## Disabilitare il sommario solo in alcuni Articulos

Per disabilitare il sommario solamente in alcuni Articulos selezionati, è necessario seguire questi due passi.

1.  Impostate il parametro `toc` a `true` in `hugo.toml`.

    ```toml
    [params]
      toc = true
    ```

2.  Aggiungete `toc = false` al front matter dell'articolo nel quale volete disabilitare il sommario.

    ```yaml
    ---
    title: Come abilitare il sommario
    date: 2023-05-02
    toc: false
    ---
    ```
