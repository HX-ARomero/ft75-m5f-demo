# useState, Context & Reducer

[Volver a Inicio](../README.md)

## ¿Cuándo es necesario un Contexto o un Reducer?

Es una de las decisiones de arquitectura más importantes en React.

Una regla práctica es pensar en **tres preguntas**:

1. **¿El estado necesita compartirse entre varios componentes?**
2. **¿La lógica para modificar ese estado es simple o compleja?**
3. **¿Necesitamos evitar pasar props a través de muchos componentes?**

---

## ¿Qué herramienta utilizar?

| Situación                                             | `useState` | Context | Reducer |
| ----------------------------------------------------- | :--------: | :-----: | :-----: |
| Estado usado por un solo componente                   |     ✅     |   ❌    |   ❌    |
| Estado local con lógica compleja                      |     ❌     |   ❌    |   ✅    |
| Estado simple compartido entre componentes            |     ❌     |   ✅    |   ❌    |
| Estado complejo compartido entre componentes          |     ❌     |   ✅    |   ✅    |
| Estado simple que se puede pasar fácilmente por props |     ✅     |   ❌    |   ❌    |
| Estado global con muchas transiciones relacionadas    |     ❌     |   ✅    |   ✅    |

---

## `useState`

Úsalo cuando el estado:

- pertenece a un componente;
- tiene pocas variables;
- tiene actualizaciones simples;
- no necesita ser compartido ampliamente.

Por ejemplo:

```tsx
const [isOpen, setIsOpen] = useState(false);
```

---

## Context

Úsalo cuando:

- varios componentes necesitan acceder al mismo estado;
- pasar ese estado mediante props se vuelve incómodo;
- el estado pertenece conceptualmente a una sección de la aplicación.

Por ejemplo:

```text
ProductsProvider
    │
    ├── SearchBar
    ├── CategoryFilter
    ├── ProductsList
    └── Pagination
```

Todos pueden acceder al mismo estado mediante `useProducts()` sin tener que pasar props entre ellos.

> **Importante:** Context no administra por sí mismo la lógica del estado. Principalmente permite **compartir valores y acciones** entre componentes.

---

## `useReducer`

Úsalo cuando el problema principal sea la **complejidad de las transiciones del estado**.

Por ejemplo, cuando tienes muchas variables relacionadas:

```text
products
isLoading
isLoadingMore
error
cursor
hasNextPage
```

y diferentes acciones pueden modificar varias de ellas simultáneamente:

```text
LOAD_START
LOAD_SUCCESS
LOAD_ERROR
LOAD_MORE_START
LOAD_MORE_SUCCESS
RESET
```

En esos casos, `useReducer` puede hacer que las transiciones sean más predecibles y centralizadas.

---

## ¿Cuándo usar ambos?

Cuando tienes **estado complejo que además necesita ser compartido**.

Es un escenario habitual para combinar:

```text
Context + useReducer
```

Por ejemplo:

```text
ProductsProvider
       │
       ├── useReducer
       │      ↓
       │   administra el estado
       │
       └── Context
              ↓
       comparte estado + acciones
              │
       ┌──────┼──────┐
       ↓      ↓      ↓
    Search  List   Pagination
```

El `Reducer` decide **cómo cambia el estado**.

El `Context` decide **quién puede acceder a ese estado y a esas acciones**.

---

## ¿Cuándo no necesitas ninguno?

Cuando el estado es local y no necesita compartirse:

```tsx
function ProductForm() {
  const [name, setName] = useState("");
  const [price, setPrice] = useState(0);

  // ...
}
```

No necesitas:

```text
Context ❌
Reducer ❌
```

Simplemente:

```text
useState ✅
```

---

## Regla práctica para recordar

Podemos resumirlo así:

> **`useState`** resuelve dónde guardar y actualizar un estado simple.
>
> **`useReducer`** organiza cómo cambia un estado complejo.
>
> **Context** resuelve cómo compartir un estado entre componentes.

Por eso no sería correcto pensar:

```text
Context vs Reducer
```

sino:

```text
             ¿Necesito compartir?
                    │
             ┌──────┴──────┐
            NO             SÍ
             │              │
     ¿Lógica compleja?   ¿Lógica compleja?
         │       │          │        │
        NO      SÍ         NO       SÍ
         │       │          │        │
    useState  useReducer  Context  Context
                                      +
                                   useReducer
```

Y existe además un caso intermedio:

```text
Estado local
     +
Reducer
```

cuando **no necesitas Context**, pero la lógica interna del componente sí es suficientemente compleja como para beneficiarse de un reducer.

> **`useReducer`** no implica que necesites Context, y **Context** no implica que necesites **`useReducer`**.

---

## Resumen

La decisión puede pensarse en dos dimensiones independientes:

### 1. ¿El estado necesita compartirse?

- **No** → `useState` o `useReducer`.
- **Sí** → Context.

### 2. ¿La lógica del estado es compleja?

- **No** → `useState`.
- **Sí** → `useReducer`.

Por lo tanto:

| Compartir estado | Complejidad | Solución               |
| ---------------- | ----------- | ---------------------- |
| ❌ No            | 🟢 Simple   | `useState`             |
| ❌ No            | 🔴 Compleja | `useReducer`           |
| ✅ Sí            | 🟢 Simple   | `Context + useState`   |
| ✅ Sí            | 🔴 Compleja | `Context + useReducer` |

Esta última tabla resume la idea fundamental:

> **Context y Reducer no compiten entre sí. Se pueden combinar porque resuelven problemas diferentes.**

---

[Volver a Inicio](../README.md)
