# Quantum Learning: Site Shell and Chapters 1–2 Labs

Date: 2026-10-13
Status: Approved design, awaiting spec review

## Purpose

Quantum Learning is a personal study companion for *Quantum Computing for Computer Scientists* (Yanofsky & Mannucci). It is used alongside the book, so every interactive piece is labeled with the book section it supports. The site focuses on interactive visualizations, randomly generated practice problems, and (in a later sub-project) a circuit simulator. It does not reproduce the textbook: any explanatory text on the site is short and original.

The book PDF stays local and is git-ignored.

## Project decomposition

The full site covers 11 chapters, split into sub-projects that each get their own spec, plan, and implementation:

1. **Site shell + Chapters 1–2 labs** (this spec)
2. Chapters 3–4 labs: probabilistic vs. quantum systems, states, measurement
3. Circuit simulator: Chapter 5, with Chapter 6 algorithms as preset circuits
4. Later chapters (7–11), added as the reader reaches them

This sub-project's `src/math/` module is designed to become the simulator's engine in sub-project 3.

## Scope of this sub-project

In scope:
- Home page listing all 11 chapters, with Chapters 1–2 linked and 3–11 marked "coming soon"
- Chapter pages for Chapters 1 and 2 with the widgets and problem types below
- Shared complex-number and linear-algebra module
- Answer parsing and checking
- Tests, and a deploy workflow that runs lint and tests before building

Out of scope (deliberately): score tracking or progress saving, accounts, a manual theme toggle (the site follows the system color scheme), any backend, hand-written conceptual problems, and Chapters 3–11 content.

## Chapter content

Each section gets a widget, a set of generated problem types, or both.

| Section | Widget | Problem types |
|---|---|---|
| §1.1–1.2 Complex algebra | **ComplexCalculator**: enter *z* and *w*; shows *z+w*, *z−w*, *z·w*, *z/w*, conjugates, moduli | add, subtract, multiply, divide; conjugate; modulus |
| §1.3 Geometry | **ComplexPlane**: drag *z* and *w* on an SVG plane and see *z·w* (angles add, lengths multiply); toggle to show powers of *z* and the *n*-th roots of unity | Cartesian → polar; polar → Cartesian; multiply in polar form |
| §2.1–2.2 ℂⁿ | **VectorWorkbench**: small complex vectors/matrices; vector addition, scalar multiplication, and matrix × vector shown one step at a time | vector addition; scalar multiplication; matrix × vector; matrix × matrix |
| §2.3 Basis and dimension | none | express a 2D vector in a given basis |
| §2.4 Inner products | **InnerProductView**: two draggable vectors in the plane showing angle, norms, and distance | complex inner product; norm; distance |
| §2.5 Eigenvalues | **PlaneTransform**: a 2×2 real matrix warps a grid; eigenvector directions are highlighted | eigenvalues of a 2×2; "is *v* an eigenvector?" |
| §2.6 Hermitian/unitary | **MatrixInspector**: enter a matrix; shows its adjoint (†) and whether it is Hermitian and/or unitary, pointing out the failing check | compute the adjoint; classify as Hermitian / unitary / both / neither |
| §2.7 Tensor product | **KroneckerGrid**: *A ⊗ B* drawn as color-coded blocks, each block equal to *aᵢⱼ·B* | vector ⊗ vector; 2×2 ⊗ 2×2 |

### Answer entry

- Complex numbers are typed as text into `ComplexInput`. Accepted forms: `3-2i`, `-i`, `i`, `2`, `2.5i`, `0.5+1.2i`, `-2i+3`, with optional spaces between terms.
- Vectors and matrices use `MatrixInput`, a grid of `ComplexInput` cells.
- Polar answers use two fields: modulus *r*, and angle *θ* in radians. The angle accepts decimals (`0.785`) or π multiples (`pi`, `pi/4`, `-3pi/4`), and is compared modulo 2π so `pi` and `-pi` are both correct. Polar problems never use *z* = 0, whose angle is undefined.
- Yes/no and classification answers use buttons.
- Numeric answers are accepted within an absolute tolerance of ±0.01 on each real and imaginary part. A wrong answer reveals the correct one.

## Architecture

### Stack

React 19 + Vite (existing scaffold), deployed to GitHub Pages at `https://mrprotsyuk.github.io/Quantum-Learning/`.

New dependencies:
- `react-router` with `HashRouter`. GitHub Pages can't serve deep links for a single-page app, so refreshing a non-hash URL would return a 404; hash URLs avoid that with no extra setup.
- `katex`: renders math notation in labels and problem prompts
- Dev: `vitest`, `@testing-library/react`, `jsdom`

Graphics are hand-written SVG; no charting library.

### Routes

- `#/`: Home (chapter roadmap)
- `#/chapter/:n`: ChapterPage for a chapter whose status is `available`
- Anything else, including a coming-soon chapter number, redirects to `#/`

### Folder layout

```
src/
  math/
    complex.js        complex numbers as {re, im}: add, sub, mul, div, conj, abs, arg, fromPolar, toPolar, pow, rootsOfUnity
    linalg.js         vectors and matrices as arrays of complex: add, scale, matVec, matMul, adjoint, inner, norm, distance,
                      isHermitian, isUnitary, kron, eigen2x2 (real 2×2 only)
  problems/
    rng.js            seeded PRNG (mulberry32) plus helpers: randInt, pick
    parse.js          parseComplex(text) → {re, im} or null; parseAngle(text) → number or null
    check.js          checkAnswer(shape, given, expected, tol = 0.01) → boolean
    generators/
      ch1.js          problem types for Chapter 1
      ch2.js          problem types for Chapter 2
      index.js        registry: problem id → generator
  widgets/            ComplexCalculator, ComplexPlane, VectorWorkbench, InnerProductView,
                      PlaneTransform, MatrixInspector, KroneckerGrid
  components/         ComplexInput, MatrixInput, PolarInput, ProblemCard, Math (KaTeX wrapper),
                      SectionBlock, ErrorBoundary, Layout
  chapters/
    registry.js       all 11 chapters: {n, title, status: 'available' | 'coming-soon', sections: [{id, title, widget?, problemIds}]}
  pages/
    Home.jsx
    ChapterPage.jsx
```

### Interfaces

**Problem generator**
```js
{
  id: 'ch1.multiply',
  title: 'Multiply two complex numbers',
  generate(rng) → { prompt, answer, shape }
}
```
- `prompt` is a KaTeX string.
- `shape` is one of: `'complex'`, `'polar'`, `'vector'` (with length), `'matrix'` (with rows and cols), `'boolean'`, `'choice'` (with an options list).
- `answer` matches `shape` and is computed with `src/math/`.

**Chapter registry entry**
```js
{ n: 1, title: 'Complex Numbers', status: 'available',
  sections: [{ id: '1.3', title: 'The Geometry of Complex Numbers',
               widget: ComplexPlane, problemIds: ['ch1.toPolar', 'ch1.fromPolar', 'ch1.polarMultiply'] }] }
```

### Data flow

The chapter registry is the only place that decides what the site shows. Home reads it to list chapters. ChapterPage reads it and, for each section, renders a `SectionBlock` containing the widget (if any) followed by one `ProblemCard` per problem id.

`ProblemCard` flow:
1. Generate a problem with a fresh random seed.
2. Render the prompt and the input that matches its `shape`.
3. On submit, parse the input (`parse.js`).
4. Check it against `answer` (`check.js`).
5. Show ✓, or ✗ with the correct answer.
6. "New problem" generates again.

Widgets and generators both use `src/math/`, so all math is written once and verified by one test suite.

## Error handling

- **Unreadable input:** an inline hint ("Couldn't read that. Try a form like `3-2i`.") appears under the input. The answer is not marked wrong and the user can resubmit.
- **Degenerate problems:** generators use small integers (typically −5…5) so problems can be worked by hand. Divisors are never 0. Eigenvalue problems use 2×2 real matrices constructed from integer eigenvalues and eigenvectors, so the eigenvalues are always real integers. Polar problems use angles that are multiples of π/6 or π/4.
- **Undefined widget values:** for example, *z/w* with *w* = 0 displays "undefined", never `NaN` or `Infinity`.
- **Widget crashes:** each `SectionBlock` is wrapped in an `ErrorBoundary`, so a broken widget shows a small error box and the rest of the page keeps working.
- **Bad routes:** redirect to Home.

## Testing

All tests run with Vitest.

- **`math/`:** known-value tests plus identities: |zw| = |z||w|; fromPolar(toPolar(z)) ≈ z; the *n*-th roots of unity each raised to *n* ≈ 1; U·U† = I for sample unitaries; adjoint(adjoint(A)) = A; kron dimensions and block structure; eigen2x2 matches A·v = λv.
- **`parse.js`:** a table of valid inputs with their expected values (every accepted form listed above), and invalid inputs (`abc`, `3++2i`, `i2`, empty string) that return `null`; the same for `parseAngle` (`pi/4`, `-3pi/4`, `0.785` valid; `pi/`, `4pi4` invalid).
- **`check.js`:** values within the tolerance pass and values just outside it fail, for each shape; polar angles of `pi` and `-pi` are treated as equal.
- **Generators:** for each problem type, 200 problems from fixed seeds. Assert: no `NaN` or `Infinity`, no zero divisors, inputs within the stated integer range, `answer` matches `shape`, and the answer agrees with an independent recomputation.
- **Smoke test:** React Testing Library renders each available ChapterPage and asserts that every registered widget and problem card appears, without errors.
- **CI:** `.github/workflows/deploy.yml` runs `npm run lint` and `npm test` before `npm run build`, so a failing check blocks the deploy.

## Success criteria

- Home lists all 11 chapters; Chapters 1–2 link to working chapter pages.
- Every section in the Chapter content table shows its widget and problem cards.
- Every problem type generates solvable problems and correctly accepts right answers and rejects wrong ones.
- `npm test` and `npm run lint` pass locally and in CI.
- The live site deploys and works, including refreshing on a chapter URL.
