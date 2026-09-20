# Genetic timetable scheduler

This project is created to build university timetables. Every teaching assignment needs a
slot, a room and a teacher, and nothing can collide.

Timetabling is NP-hard. A genetic algorithm runs over populations of candidate timetables,
each scored by a weighted fitness function.

## Why a genetic algorithm

We have hard constraints: nobody double-booked, every assignment covered for its required
hours.

Soft constraints are preferences. We collect these from teachers and transform them into
`(teacher, day, slot, PREFERRED_FREE | PREFERRED_BUSY)` tuples. They contradict each other,
so no timetable satisfies all of them. A constraint solver wants constraints that can
actually be satisfied; here you want to lose as little as possible. Violating a preference
costs points rather than killing the candidate.

Hard constraints are handled separately, by repair rather than by penalty. `repairSchedule`
runs after initialisation, crossover and mutation, relocating any conflicting event, so the
population stays feasible and fitness only scores the things that are a matter of degree.

## The algorithm

A chromosome is a full weekly timetable: a variable-length list of events, each with a group,
subject, teacher, classroom, day and pair slot.

Selection is binary tournament. Crossover is single point over the event list, followed by
repair. Mutation picks uniformly among five operators: move an event to another weekday, move
it to another pair slot, swap day and slot between two subjects in one group, add a lesson,
delete a lesson. The last two change chromosome length, which is how coverage errors get
corrected in either direction.

```ts
interface GeneticAlgorithmConfig {
  populationSize: number;
  crossoverRate: number;
  mutationRate: number;
  generations: number;
}
```

## The fitness function

Six penalty terms, in `generation/fitnessFunction.ts`:

| Term | Penalises |
|---|---|
| Teacher gaps | idle slots between a teacher's classes on the same day |
| Group gaps | same, for student groups |
| Teacher hours | deviation from contracted load |
| Preferences | a class in a `PREFERRED_FREE` slot, or a `PREFERRED_BUSY` slot left empty |
| Classroom utilisation | small group in a large room, scaled by how far under capacity |
| Coverage | missing and over-scheduled hours per group, subject and lesson type |

## Layout

The solver lives in `src/schedules/generation/`: `geneticAlgorithm.ts` (selection, crossover,
mutation, repair, the generation loop), `fitnessFunction.ts`, `scheduleGenerator.ts` for the
initial population, and `expandWeeklySchedule.ts` to spread a weekly pattern over a semester.

Around it: domain modules for teachers, groups, classrooms, subjects, semesters and
assignments, a `teacher-preferences` module, an `excelparser` for importing existing
timetables, and auth.

NestJS and TypeScript, Prisma over PostgreSQL, OpenAPI spec in `openapi.yaml`.

## Tests

`geneticAlgorithm.spec.ts` is bigger than the file it tests. The solver is stochastic, so the
tests check invariants instead of outputs: no hard constraint violated, fitness improving
across generations, coverage arithmetic correct.

## Running it

```bash
cp .env.example .env
docker compose up -d
npm install
npx prisma migrate deploy
npm run start:dev
```
