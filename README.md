# arquitect — agente de OpenCode

Agente primario para OpenCode V2 que **analiza el proyecto, ejecuta comandos y
consulta la base de datos**, pero **no escribe código**. Su trabajo es decirte
exactamente qué fragmento pegar, dónde va, qué hace y qué problema resuelve. Tú
decides qué copiar.

## Qué hace

- Lee y analiza todo el proyecto: código, arquitectura, errores, tests.
- Ejecuta comandos: shell, `git`, builds, tests, linters y CLIs de base de datos.
- Consulta la base de datos en **solo lectura** (`SELECT`, `SHOW`, `DESCRIBE`,
  `EXPLAIN`, `PRAGMA`, `.tables`, `.schema`).
- Entrega cada cambio como un fragmento completo, copiable, con la ruta exacta,
  la ubicación (línea / función / bloque) y una explicación.

## Qué no hace

- No modifica archivos del proyecto: `edit`, `write` y `patch` están denegados.
- No aplica cambios en la base de datos.
- No hace `commit`, `push` ni cambios de rama (sí puede inspeccionar con `git`).

## Instalación

Copia el agente a tu configuración global:

```sh
mkdir -p ~/.config/opencode/agents
cp opencode/agents/arquitect.md ~/.config/opencode/agents/arquitect.md
```

O instálalo solo para un proyecto:

```sh
mkdir -p .opencode/agents
cp opencode/agents/arquitect.md .opencode/agents/arquitect.md
```

OpenCode detecta el archivo al guardarlo. Si el selector no lo muestra, ejecuta
`/reload`.

## Uso

1. `Shift+Tab` hasta `arquitect`, o `/agents` y selecciónalo.
2. Pídele el cambio: *"agrega validación de email al registro"*.
3. Te responde con el fragmento, la ubicación y el porqué.
4. Lo pegas tú.
5. Le dices *"listo"* y relee el archivo para verificar antes de seguir.

## Formato de respuesta

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

## Permisos

| Acción | Efecto | Detalle |
| --- | --- | --- |
| `shell` | allow | Igual que el agente Build: `git`, builds, tests, linters, DB |
| `edit` | deny | No puede escribir archivos |
| `shell` (escrituras SQL) | ask | `DELETE`, `INSERT`, `UPDATE`, `DROP`, `ALTER`, `TRUNCATE`, `CREATE`, `GRANT` |
| `read *.env` | ask | Política base de OpenCode (hereda el prompt de secretos) |

Las reglas SQL se evalúan sobre el texto del comando, en minúsculas y mayúsculas.

## Limitaciones

- **El "solo SELECT" no es una garantía dura.** Es una instrucción del prompt más
  reglas `ask` sobre los verbos más comunes. SQL ofuscado o dentro de un archivo
  ejecutado con `psql -f` no se detecta. Para blindaje real, usa un usuario de base
  de datos con permisos de solo lectura.
- El shell tiene permisos amplios, así que técnicamente podría escribir archivos
  con redirecciones (`>`, `tee`). La restricción es de comportamiento, no de
  sandbox. Si necesitas bloqueo estricto, cambia el `allow` de shell por una
  lista blanca de comandos.
- Los archivos `.env` siguen pidiendo aprobación al leerse.

## Estructura

```
opencode/agents/arquitect.md   # definición del agente
README.md                 # este archivo
```
