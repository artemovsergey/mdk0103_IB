# Практическая работа 13. Асинхронность. Разработка многопоточного приложения

Асинхронные операции в React Native выполняются с помощью **`async`/`await`** и промисов. В этой работе вы примените эти знания в приложении и научитесь обновлять интерфейс во время длительных операций, не блокируя его.

### Пререквизиты

- Знание основ JavaScript/TypeScript, включая функции и стрелочные функции.
- Умение создавать интерфейс в React Native.
- Базовое понимание промисов и `async`/`await`.

### Что вы узнаете

- Как выполнять отложенные операции через `async` и `await`.
- Как обновлять состояние во время выполнения цикла, не блокируя интерфейс.
- Как организовать паузу и сброс длительной операции.
- Как писать тесты для асинхронных функций.

### Что вы создадите

Приложение **Race Tracker**, которое имитирует гонку двух игроков. У приложения две кнопки — `Старт/Пауза` и `Сброс` — и два индикатора прогресса.

## Обзор приложения

Игроки 1 и 2 «бегут» одновременно с разной скоростью: игрок 2 движется вдвое быстрее игрока 1. Прогресс каждого отображается индикатором. Кнопка `Старт/Пауза` запускает, приостанавливает и продолжает гонку, `Сброс` возвращает оба прогресса к нулю.

## Как это работает в React Native

В React Native асинхронную функцию объявляют через `async`, а паузу без блокировки выполняют с помощью `setTimeout` внутри промиса:

Функция «бега» игрока:

```tsx
function delay(ms: number) {
  return new Promise((resolve) => setTimeout(resolve, ms))
}

async function run(
  update: (progress: number) => void,
  maxProgress: number,
  delayMillis: number,
  isPaused: () => boolean,
) {
  let current = 0
  while (current < maxProgress) {
    if (isPaused()) {
      await delay(100)
      continue
    }
    await delay(delayMillis)
    current += 1
    update(current)
  }
}
```

- `while` выполняется, пока прогресс не достигнет `maxProgress` (100).
- `await delay(...)` приостанавливает функцию, не блокируя интерфейс.
- `isPaused()` позволяет приостановить гонку.

## Компонент RaceParticipant

```tsx
type RaceParticipantProps = {
  name: string
  maxProgress: number
  delayMillis: number
  progress: number
}

function RaceParticipant({ name, maxProgress, delayMillis, progress }: RaceParticipantProps) {
  return (
    <View style={styles.participant}>
      <Text style={styles.name}>{name}</Text>
      <View style={styles.track}>
        <View
          style={[
            styles.fill,
            { width: `${(progress / maxProgress) * 100}%` },
          ]}
        />
      </View>
      <Text>{progress} / {maxProgress}</Text>
    </View>
  )
}
```

Полоса прогресса — это `View` с шириной в процентах. Она перерисовывается при изменении `progress`.

## Главный компонент

```tsx
export default function App() {
  const MAX_PROGRESS = 100

  const [player1, setPlayer1] = useState(0)
  const [player2, setPlayer2] = useState(0)
  const [running, setRunning] = useState(false)

  const pausedRef = useRef(false)

  const startRace = async () => {
    setRunning(true)
    pausedRef.current = false

    const p1 = run(setPlayer1, MAX_PROGRESS, 300, () => pausedRef.current)
    const p2 = run(setPlayer2, MAX_PROGRESS, 150, () => pausedRef.current)

    await Promise.all([p1, p2])
    setRunning(false)
  }

  const togglePause = () => {
    pausedRef.current = !pausedRef.current
  }

  const reset = () => {
    pausedRef.current = false
    setRunning(false)
    setPlayer1(0)
    setPlayer2(0)
  }

  return (
    <View style={styles.container}>
      <RaceParticipant name="Игрок 1" maxProgress={MAX_PROGRESS} delayMillis={300} progress={player1} />
      <RaceParticipant name="Игрок 2" maxProgress={MAX_PROGRESS} delayMillis={150} progress={player2} />

      <Button
        title={running ? (pausedRef.current ? 'Продолжить' : 'Пауза') : 'Старт'}
        onPress={running ? togglePause : startRace}
      />
      <Button title="Сброс" onPress={reset} />
    </View>
  )
}
```

Важные моменты:

- `Promise.all([p1, p2])` запускает обоих бегунов **параллельно** и ждёт завершения обоих.
- `pausedRef` — это `useRef`: его значение не вызывает перерисовку и доступно внутри асинхронного цикла.
- Игрок 2 использует вдвое меньшую задержку (150 мс против 300 мс), поэтому бежит вдвое быстрее.

## Полный код

```tsx
import { StatusBar } from 'expo-status-bar';
import { useRef, useState } from 'react';
import { Button, StyleSheet, Text, View } from 'react-native';

function delay(ms: number) {
  return new Promise((resolve) => setTimeout(resolve, ms))
}

async function run(
  update: (progress: number) => void,
  maxProgress: number,
  delayMillis: number,
  isPaused: () => boolean,
) {
  let current = 0
  while (current < maxProgress) {
    if (isPaused()) {
      await delay(100)
      continue
    }
    await delay(delayMillis)
    current += 1
    update(current)
  }
}

function RaceParticipant({
  name,
  maxProgress,
  progress,
}: {
  name: string
  maxProgress: number
  progress: number
}) {
  return (
    <View style={styles.participant}>
      <Text style={styles.name}>{name}</Text>
      <View style={styles.track}>
        <View style={[styles.fill, { width: `${(progress / maxProgress) * 100}%` }]} />
      </View>
      <Text>{progress} / {maxProgress}</Text>
    </View>
  )
}

export default function App() {
  const MAX_PROGRESS = 100

  const [player1, setPlayer1] = useState(0)
  const [player2, setPlayer2] = useState(0)
  const [running, setRunning] = useState(false)

  const pausedRef = useRef(false)

  const startRace = async () => {
    setRunning(true)
    pausedRef.current = false
    await Promise.all([
      run(setPlayer1, MAX_PROGRESS, 300, () => pausedRef.current),
      run(setPlayer2, MAX_PROGRESS, 150, () => pausedRef.current),
    ])
    setRunning(false)
  }

  const togglePause = () => {
    pausedRef.current = !pausedRef.current
  }

  const reset = () => {
    pausedRef.current = false
    setRunning(false)
    setPlayer1(0)
    setPlayer2(0)
  }

  return (
    <View style={styles.container}>
      <RaceParticipant name="Игрок 1" maxProgress={MAX_PROGRESS} progress={player1} />
      <RaceParticipant name="Игрок 2" maxProgress={MAX_PROGRESS} progress={player2} />
      <View style={styles.buttons}>
        <Button
          title={running ? (pausedRef.current ? 'Продолжить' : 'Пауза') : 'Старт'}
          onPress={running ? togglePause : startRace}
        />
        <Button title="Сброс" onPress={reset} />
      </View>
      <StatusBar style="auto" />
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    padding: 24,
  },
  participant: { width: '100%', marginBottom: 24 },
  name: { fontSize: 18, marginBottom: 8 },
  track: {
    height: 16,
    backgroundColor: '#E0E0E0',
    borderRadius: 8,
    overflow: 'hidden',
  },
  fill: { height: '100%', backgroundColor: '#3F51B5' },
  buttons: { width: '100%', gap: 8, marginTop: 16 },
})
```

## Тестирование асинхронной функции

Асинхронные функции тестируют с помощью `async`/`await` в тесте. Пример на Jest:

```tsx
test('run достигает максимума', async () => {
  let progress = 0
  await run((p) => { progress = p }, 5, 1, () => false)
  expect(progress).toBe(5)
})
```

## Итог

- `async`/`await` и `Promise.all` запускают функции параллельно.
- `await new Promise(r => setTimeout(r, ms))` — пауза без блокировки интерфейса.
- `useState` обновляет интерфейс без блокировки.
- `useRef` хранит изменяемое значение между перерисовками (например, флаг паузы).
