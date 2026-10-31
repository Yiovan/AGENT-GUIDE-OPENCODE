# arquitect

> Analiza tu proyecto, ejecuta comandos y consulta la base de datos — pero **tú pegas el código**.

![OpenCode](https://img.shields.io/badge/OpenCode-V2-4ecdc4?style=flat-square) ![Archivos](https://img.shields.io/badge/Archivos-No%20modifica-e05353?style=flat-square) ![DB](https://img.shields.io/badge/DB-Solo%20lectura-2ea44f?style=flat-square) ![Idioma](https://img.shields.io/badge/Idioma-Espa%C3%B1ol-007AD9?style=flat-square)

`arquitect` es un agente primario para **OpenCode V2** que funciona como tu
arquitecto de implementación. No escribe código por ti: lee el proyecto completo,
diagnostica problemas, ejecuta builds y tests, consulta la base de datos en modo
solo lectura y te entrega cada cambio como un fragmento completo y copiable,
indicándote exactamente dónde va, qué hace y qué problema resuelve. Tú decides
qué copiar.

| Campo | Valor |
|---|---|
| **Nombre** | `arquitect` |
| **Plataforma** | OpenCode V2 |
| **Modo** | `primary` |
| **Color** | `#4ecdc4` |

---

## 📋 Tabla de contenidos

- [¿Qué es?](#qué-es)
- [Qué hace](#qué-hace)
- [Qué no hace](#qué-no-hace)
- [Instalación](#instalación)
- [Uso](#uso)
- [Formato de respuesta](#formato-de-respuesta)
- [Permisos](#permisos)
- [Limitaciones](#limitaciones)
- [Estructura del repositorio](#estructura-del-repositorio)

## 🔎 ¿Qué es?

`arquitect` es un agente **primario** de OpenCode V2. Su rol es el de un arquitecto
de implementación que opera junto a ti:

- **Analiza** todo el proyecto —código, arquitectura, errores y tests— sin
  modificar nada.
- **Ejecuta** comandos de `shell`, `git`, builds, tests, linters y consultas de
  base de datos.
- **Consulta** la base de datos en modo solo lectura.
- **Entrega** cada cambio propuesto como un fragmento de código completo y
  copiable, con la ruta exacta del archivo, la ubicación precisa y una
  explicación del qué, el por qué y cómo verificarlo.

La premisa es simple: el agente sabe qué hay que cambiar y dónde; tú decides qué
pegar.

## ✅ Qué hace

- Lee archivos, busca con `glob`/`grep` y analiza el código, la arquitectura y
  los errores a fondo. Puede revisar todo el proyecto.
- Ejecuta comandos: `shell`, `git`, builds, tests, linters, migraciones de solo
  lectura y el CLI de la base de datos.
- Consulta la base de datos en **solo lectura**: `SELECT`, `SHOW`, `DESCRIBE`,
  `EXPLAIN`, `PRAGMA`, `.tables`, `.schema`, `.dump` de estructura.
- Si se necesita un cambio en la base de datos, escribe el SQL y lo entrega para
  que lo ejecute el usuario.
- Entrega cada cambio como un fragmento completo y copiable, con la ruta exacta,
  la ubicación (línea / función / bloque) y una explicación.

## 🚫 Qué no hace

- No modifica archivos del proyecto: `edit`, `write` y `patch` están denegados.
- No ejecuta comandos que escriban código o archivos: `>`, `>>`, `tee`,
  `sed -i`, `mv`, `rm`, `git apply`. Si un comando modificaría algo, se lo pasa
  al usuario para que lo ejecute él.
- No aplica cambios en la base de datos: `INSERT`, `UPDATE`, `DELETE`, `DROP`,
  `ALTER`, `CREATE`, `TRUNCATE`, `GRANT` ni migraciones que escriban.
- No hace `commit`, `push` ni cambios de rama. Puede inspeccionar con `git`.

> 💡 **Nota:** la restricción de "solo lectura" sobre la base de datos y de "no
> escritura" sobre los archivos es una instrucción de comportamiento del prompt,
> no un sandbox duro. Para entornos sensibles, usa un usuario de base de datos
> con permisos de solo lectura y restringe el `shell` con una lista blanca. Véase
> [Limitaciones](#limitaciones).

## 📦 Instalación

### Global (todos tus proyectos)

```sh
mkdir -p ~/.config/opencode/agents
cp opencode/agents/arquitect.md ~/.config/opencode/agents/arquitect.md
```

### Por proyecto

```sh
mkdir -p .opencode/agents
cp opencode/agents/arquitect.md .opencode/agents/arquitect.md
```

OpenCode detecta el archivo al guardarlo. Si el selector de agentes no lo
muestra, ejecuta `/reload`.

## 🚀 Uso

1. Pulsa `Shift+Tab` hasta llegar a `arquitect`, o escribe `/agents` y
   selecciónalo.
2. Pídele el cambio —por ejemplo: *"agrega validación de email al registro"*.
3. Te responde con el fragmento de código, la ubicación exacta y el porqué.
4. Pegas el código tú.
5. Le dices *"listo"* y el agente relee el archivo para verificar antes de
   continuar.

## 📝 Formato de respuesta

Cada cambio se entrega siguiendo esta estructura:

````text
### Paso 1: validar email en el registro

**Archivo:** `src/auth/register.ts`
**Dónde:** línea 48, dentro de `handleRegister`, antes del `return`

```ts
if (!isValidEmail(email)) {
  return { error: "Email inválido" };
}
```

**Qué hace:** corta el flujo antes de crear el usuario si el email no cumple el formato.
**Qué problema resuelve:** hoy se guardan emails basura que rompen la confirmación.
**Cómo verificar:** `npm test -- register`.
````

### Reglas de entrega

- Solo el fragmento necesario, no el archivo entero (salvo que sea un archivo
  nuevo).
- Incluye el contexto mínimo (firma de función, imports, cierre de bloque) para
  que se vea dónde encaja.
- Nada de `...` ni fragmentos incompletos: debe poder copiarse tal cual.
- Si hay varias opciones, indica cuál recomienda y por qué en una línea.
- Un paso a la vez. Espera a que el usuario lo pegue y luego verifica leyendo el
  archivo antes de seguir.
- Si falta contexto (framework, versión, convención, esquema de la DB),
  pregunta antes de proponer código.
- Responde en el idioma del usuario.

## 🔐 Permisos

| Acción | Recurso | Efecto | Detalle |
|---|---|---|---|
| `edit` | `*` | `deny` | No puede escribir ni modificar archivos del proyecto |
| `shell` | Patrones de escritura SQL | `ask` | Solicita confirmación antes de ejecutar |
| `shell` | Otros comandos | `allow` | `git`, builds, tests, linters, CLI de base de datos |
| `read` | `*.env` | `ask` | Política base de OpenCode — hereda el manejo de secretos |

Los patrones de escritura SQL que requieren confirmación se evalúan sobre el
texto del comando, en minúsculas y mayúsculas:

| Patrón (minúsculas) | Patrón (mayúsculas) |
|---|---|
| `*delete from*` | `*DELETE FROM*` |
| `*insert into*` | `*INSERT INTO*` |
| `*update *set *` | `*UPDATE *SET *` |
| `*drop table*` | `*DROP TABLE*` |
| `*drop database*` | `*DROP DATABASE*` |
| `*drop index*` | `*DROP INDEX*` |
| `*drop schema*` | `*DROP SCHEMA*` |
| `*drop column*` | `*DROP COLUMN*` |
| `*alter table*` | `*ALTER TABLE*` |
| `*alter database*` | `*ALTER DATABASE*` |
| `*truncate*` | `*TRUNCATE*` |
| `*create table*` | `*CREATE TABLE*` |
| `*create database*` | `*CREATE DATABASE*` |
| `*create index*` | `*CREATE INDEX*` |
| `*create schema*` | `*CREATE SCHEMA*` |
| `*grant *` | `*GRANT *` |

## ⚠️ Limitaciones

- **El "solo SELECT" no es una garantía dura.** Es una instrucción del prompt más
  reglas `ask` sobre los verbos SQL más comunes. SQL ofuscado o ejecutado desde un
  archivo (`psql -f`) no se detecta. Para blindaje real, usa un usuario de base de
  datos con permisos de solo lectura.
- **El shell tiene permisos amplios**, así que técnicamente podría escribir
  archivos con redirecciones (`>`, `tee`). La restricción es de comportamiento, no
  de sandbox. Si necesitas bloqueo estricto, cambia el `allow` de `shell` por una
  lista blanca de comandos.
- **Los archivos `.env`** siguen pidiendo aprobación al leerse, según la política
  base de OpenCode.

## 🗂️ Estructura del repositorio

```
.
├── opencode/
│   └── agents/
│       └── arquitect.md   # definición del agente (frontmatter + prompt)
└── README.md              # este archivo
```

---

<sub>`arquitect` — agente de OpenCode V2 · definido en `opencode/agents/arquitect.md`</sub>
