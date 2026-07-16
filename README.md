# 🧪 Laboratorio 1: Introducción a ANTLR y Analisis del mismo

## 📋 Descripción General

En este laboratorio trabajarás con **ANTLR**, un generador de analizadores sintácticos. Hemos proporcionado un `Dockerfile` para ayudarte a configurar el entorno rápidamente. Utilizaremos Python para hacer pruebas, ya que es más sencillo que Java para pruebas pequeñas.

* **Modalidad: Individual**

## 🧰 Instrucciones de Configuración

1. **Construir y Ejecutar el Contenedor Docker**Desde el directorio raíz de este laboratorio, ejecuta el siguiente comando para construir la imagen y lanzar un contenedor interactivo:

   ```bash
   docker build -t lab1-image .
   docker run --rm -ti -v "${PWD}\program:/program" lab1-image
   ```
2. **Entender el Entorno**

   - El directorio `program` se monta dentro del contenedor.
   - Este contiene la **gramática de ANTLR**, un archivo `Driver.py` (punto de entrada principal) y un archivo `program_test.txt` (entrada de prueba).
3. **Generar Archivos de Lexer y Parser**Dentro del contenedor, compila la gramática ANTLR a Python con:

   ```bash
   antlr -Dlanguage=Python3 MiniLang.g4
   ```
4. **Ejecutar el Analizador**
   Usa el driver para analizar el archivo de prueba:

   ```bash
   python3 Driver.py program_test.txt
   ```

   - ✅ Si el archivo es sintácticamente correcto, **no se mostrará ningún resultado**.
   - ❌ Si existen errores, ANTLR los mostrará en la consola.
   - **Next Step:** Jueguen editando el archivo y vean los cambios en los resultados de compilación.

# Análisis: Gramática ANTLR y Driver.py - MiniLang

Este documento cubre el entregable de análisis del laboratorio: cómo está estructurada `MiniLang.g4` y cómo funciona `Driver.py`.

## 1. Estructura de un archivo `.g4`

Un archivo de gramática ANTLR se organiza en tres bloques:

1. **Declaración de la gramática**: `grammar MiniLang;` — el nombre debe coincidir con el nombre del archivo (`MiniLang.g4`). ANTLR usa este nombre para generar las clases `MiniLangLexer` y `MiniLangParser`.
2. **Reglas del parser** (empiezan con minúscula): definen la estructura sintáctica del lenguaje — cómo se combinan los tokens en sentencias y expresiones.
3. **Reglas del lexer** (empiezan con mayúscula): definen cómo se agrupan los caracteres de entrada en *tokens* (las unidades léxicas que después consume el parser).

ANTLR genera automáticamente el lexer y el parser en el lenguaje indicado (`-Dlanguage=Python3` en este caso) a partir de estas reglas.

## 2. Reglas del parser

```antlr
prog:   stat+ ;

stat:   expr NEWLINE                 # printExpr
    |   ID '=' expr NEWLINE          # assign
    |   NEWLINE                      # blank
    ;

expr:   expr ('*'|'/') expr          # MulDiv
    |   expr ('+'|'-') expr          # AddSub
    |   INT                          # int
    |   ID                           # id
    |   '(' expr ')'                 # parens
    ;
```

- **`prog`** es la regla inicial (la que invoca `Driver.py` con `parser.prog()`). Un programa es una o más sentencias (`stat+`).
- **`stat`** define tres formas válidas de sentencia:
  - una expresión seguida de salto de línea (`printExpr`),
  - una asignación `ID '=' expr` (`assign`),
  - o una línea vacía (`blank`).
- **`expr`** es una regla **recursiva a la izquierda** (`expr: expr op expr | ...`). ANTLR 4 soporta recursión izquierda directamente y la reescribe internamente en un parser eficiente; no hay que factorizarla a mano como en LL(1) clásico.
- **El orden de las alternativas en `expr` define la precedencia**: las alternativas declaradas primero tienen mayor precedencia. Por eso `MulDiv` está antes que `AddSub` → `*` y `/` se evalúan antes que `+` y `-`, igual que en la aritmética estándar.
- La asociatividad por defecto en ANTLR para reglas recursivas es **izquierda**, por lo que `a - b - c` se agrupa como `(a - b) - c`.

## 3. El uso de `#` (etiquetas de alternativa)

Cada `# Nombre` al final de una alternativa es una **etiqueta**. Le dice a ANTLR que genere un contexto de árbol (`MulDivContext`, `AssignContext`, `IntContext`, etc.) y un método visitor/listener independiente para *esa alternativa específica*, en lugar de un único método genérico para toda la regla (`StatContext`, `ExprContext`).

Esto es clave quando luego se implementen **Visitors** o **Listeners** (siguiente etapa del curso): en vez de tener un solo `visitStat()` con un `if/else` gigante para distinguir el caso, ANTLR genera `visitPrintExpr()`, `visitAssign()`, `visitBlank()`, `visitMulDiv()`, `visitAddSub()`, `visitInt()`, `visitId()`, `visitParens()` por separado, cada uno tipado a su propio contexto.

## 4. Reglas del lexer

```antlr
MUL : '*' ;
DIV : '/' ;
ADD : '+' ;
SUB : '-' ;
ID  : [a-zA-Z]+ ;
INT : [0-9]+ ;
NEWLINE:'\r'? '\n' ;
WS  : [ \t]+ -> skip ;
```

- Cada regla en mayúsculas define un **token**. Los literales (`'*'`, `'+'`, etc.) usados directamente en las reglas del parser (como `'(' expr ')'`) también generan tokens implícitos.
- **El orden importa**: cuando dos reglas podrían matchear la misma entrada con la misma longitud, gana la que está declarada primero en el archivo. Cuando el largo del match difiere, ANTLR usa *maximal munch* (siempre prefiere el match más largo posible) — por eso `ID : [a-zA-Z]+` puede consumir `abc` completo como un solo token en vez de tres tokens de una letra.
- `NEWLINE` captura el fin de línea (`\r\n` o `\n`) y se envía al parser como una señal de fin de sentencia — es parte de la gramática, no se descarta.
- `WS -> skip` descarta espacios y tabulaciones: el lexer los reconoce pero **no** los envía como tokens al parser, así el parser nunca tiene que lidiar con espacios en blanco.
- No hay una regla para comentarios ni para otros caracteres (p. ej. `#`, `@`, `%`); si aparecen en la entrada, el lexer no puede tokenizarlos y reporta un error léxico.

## 5. `Driver.py` — funcionamiento

```python
input_stream = FileStream(argv[1])
lexer = MiniLangLexer(input_stream)
stream = CommonTokenStream(lexer)
parser = MiniLangParser(stream)
tree = parser.prog()
```

El flujo es el pipeline estándar de ANTLR en tiempo de ejecución:

1. **`FileStream(argv[1])`**: lee el archivo de entrada (pasado como argumento por línea de comandos) como stream de caracteres.
2. **`MiniLangLexer(input_stream)`**: el analizador léxico generado por ANTLR consume el stream de caracteres y produce tokens (`ID`, `INT`, `MUL`, `NEWLINE`, etc.) según las reglas léxicas de la gramática.
3. **`CommonTokenStream(lexer)`**: envuelve al lexer en un stream de tokens que el parser puede consumir bajo demanda (y sobre el cual puede hacer *lookahead*).
4. **`MiniLangParser(stream)`**: el analizador sintáctico generado, que construye el árbol de derivación a partir de los tokens.
5. **`parser.prog()`**: invoca la **regla inicial** de la gramática (debe coincidir con alguna regla del parser; en este caso `prog`, la raíz definida en la gramática) y devuelve el árbol sintáctico (`ParseTree`) resultante.

`Driver.py` no imprime nada por sí mismo: solo construye el árbol. Por defecto, ANTLR usa el `DefaultErrorListener`, que **escribe los errores léxicos y sintácticos a stderr automáticamente** sin necesidad de código adicional — por eso el laboratorio indica que si el programa es correcto no se ve ninguna salida, y si hay errores, ANTLR los reporta solo. Si se quisiera hacer algo con el árbol (imprimirlo, recorrerlo, evaluarlo), habría que agregar un Visitor o Listener explícito — eso es justamente lo que se hará más adelante en el curso.

## 📹 Video de Referencia

[Video explicativo del laboratorio](https://youtu.be/kfx1FmLxosc)
