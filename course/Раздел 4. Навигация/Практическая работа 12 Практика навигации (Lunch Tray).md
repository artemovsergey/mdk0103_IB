# Практическая работа 12. Навигация (Lunch Tray)

В этой работе вы закрепите навыки навигации из лекции 9, создав приложение **Lunch Tray** — пошаговый заказ обеда.

### Пререквизиты

- Лекция 9 «Навигация».
- Умение настроить `NavigationContainer` и `Stack.Navigator`.
- Умение передавать параметры между экранами.

### Что вы узнаете

- Как организовать пошаговый поток из нескольких экранов.
- Как передавать состояние между экранами через параметры.
- Как реализовать отмену заказа и возврат к началу.
- Как динамически менять заголовок экрана.

### Что вы создадите

Приложение **Lunch Tray** с пятью экранами: выбор основного блюда, гарнира, напитка, дополнительного блюда и итоговый чек.

## Обзор потока экранов

```
Start → Entree → SideDish → Accompaniment → Checkout
```

На каждом экране пользователь выбирает один вариант и нажимает «Далее». На экране `Checkout` показывается итог и кнопка «Отправить заказ». Кнопка «Отмена» в заголовке возвращает к началу.

## Установка

```
npx expo install @react-navigation/native @react-navigation/native-stack
npx expo install react-native-screens react-native-safe-area-context
```

## Общий компонент выбора

Все экраны выбора похожи. Создадим переиспользуемый компонент:

```tsx
function SelectionScreen({
  title,
  options,
  onNext,
}: {
  title: string
  options: string[]
  onNext: (option: string) => void
}) {
  const [selected, setSelected] = useState<string | null>(null)

  return (
    <View style={styles.container}>
      <Text style={styles.title}>{title}</Text>
      {options.map((option) => (
        <Button
          key={option}
          title={selected === option ? `✓ ${option}` : option}
          onPress={() => setSelected(option)}
        />
      ))}
      <Button
        title="Далее"
        disabled={selected === null}
        onPress={() => onNext(selected!)}
      />
    </View>
  )
}
```

## Экраны приложения

```tsx
import { NavigationContainer } from '@react-navigation/native'
import { createNativeStackNavigator } from '@react-navigation/native-stack'

const Stack = createNativeStackNavigator()

function EntreeScreen({ navigation }: { navigation: any }) {
  return (
    <SelectionScreen
      title="Выберите основное блюдо"
      options={['Сэндвич с индейкой', 'Овощная лазанья', 'Салат с тунцом']}
      onNext={(entree) => navigation.navigate('SideDish', { entree })}
    />
  )
}

function SideDishScreen({ navigation, route }: { navigation: any; route: any }) {
  return (
    <SelectionScreen
      title="Выберите гарнир"
      options={['Картофель фри', 'Салат', 'Суп']}
      onNext={(sideDish) =>
        navigation.navigate('Accompaniment', { ...route.params, sideDish })
      }
    />
  )
}

function AccompanimentScreen({ navigation, route }: { navigation: any; route: any }) {
  return (
    <SelectionScreen
      title="Выберите напиток"
      options={['Чай', 'Кофе', 'Лимонад']}
      onNext={(accompaniment) =>
        navigation.navigate('Checkout', { ...route.params, accompaniment })
      }
    />
  )
}
```

Обратите внимание на приём `{ ...route.params, sideDish }`: так мы **накапливаем** ранее выбранные блюда и передаём их дальше. К моменту выхода на экран `Checkout` в `route.params` собраны все выборы.

## Экран Checkout

```tsx
function CheckoutScreen({ route }: { route: any }) {
  const { entree, sideDish, accompaniment } = route.params

  const shareOrder = async () => {
    const { Share } = require('react-native')
    await Share.share({
      message: `Заказ: ${entree}, ${sideDish}, ${accompaniment}`,
    })
  }

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Ваш заказ</Text>
      <Text>Основное: {entree}</Text>
      <Text>Гарнир: {sideDish}</Text>
      <Text>Напиток: {accompaniment}</Text>
      <Button title="Отправить заказ" onPress={shareOrder} />
    </View>
  )
}
```

## Корневой компонент и заголовки

Заголовок каждого экрана задаётся через `options.title`. Кнопку «Отмена» добавляют в `headerRight` или `headerLeft`:

```tsx
export default function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator initialRouteName="Entree">
        <Stack.Screen
          name="Entree"
          component={EntreeScreen}
          options={({ navigation }) => ({
            title: 'Lunch Tray',
            headerBackVisible: false,
          })}
        />
        <Stack.Screen name="SideDish" component={SideDishScreen} options={{ title: 'Гарнир' }} />
        <Stack.Screen
          name="Accompaniment"
          component={AccompanimentScreen}
          options={{ title: 'Напиток' }}
        />
        <Stack.Screen name="Checkout" component={CheckoutScreen} options={{ title: 'Чек' }} />
      </Stack.Navigator>
    </NavigationContainer>
  )
}
```

- `headerBackVisible: false` скрывает кнопку «Назад» на первом экране.
- На остальных экранах кнопка «Назад» появляется автоматически — это и есть отмена текущего шага.

Чтобы вернуться к самому началу из любого места, используйте:

```tsx
navigation.popToTop()
```

## Полный код

```tsx
import { StatusBar } from 'expo-status-bar';
import { useState } from 'react';
import { Button, Share, StyleSheet, Text, View } from 'react-native';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

function SelectionScreen({
  title,
  options,
  onNext,
}: {
  title: string
  options: string[]
  onNext: (option: string) => void
}) {
  const [selected, setSelected] = useState<string | null>(null)
  return (
    <View style={styles.container}>
      <Text style={styles.title}>{title}</Text>
      {options.map((option) => (
        <Button
          key={option}
          title={selected === option ? `✓ ${option}` : option}
          onPress={() => setSelected(option)}
        />
      ))}
      <Button
        title="Далее"
        disabled={selected === null}
        onPress={() => onNext(selected!)}
      />
    </View>
  )
}

function EntreeScreen({ navigation }: { navigation: any }) {
  return (
    <SelectionScreen
      title="Выберите основное блюдо"
      options={['Сэндвич с индейкой', 'Овощная лазанья', 'Салат с тунцом']}
      onNext={(entree) => navigation.navigate('SideDish', { entree })}
    />
  )
}

function SideDishScreen({ navigation, route }: { navigation: any; route: any }) {
  return (
    <SelectionScreen
      title="Выберите гарнир"
      options={['Картофель фри', 'Салат', 'Суп']}
      onNext={(sideDish) =>
        navigation.navigate('Accompaniment', { ...route.params, sideDish })
      }
    />
  )
}

function AccompanimentScreen({ navigation, route }: { navigation: any; route: any }) {
  return (
    <SelectionScreen
      title="Выберите напиток"
      options={['Чай', 'Кофе', 'Лимонад']}
      onNext={(accompaniment) =>
        navigation.navigate('Checkout', { ...route.params, accompaniment })
      }
    />
  )
}

function CheckoutScreen({ route }: { route: any }) {
  const { entree, sideDish, accompaniment } = route.params
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Ваш заказ</Text>
      <Text>Основное: {entree}</Text>
      <Text>Гарнир: {sideDish}</Text>
      <Text>Напиток: {accompaniment}</Text>
      <Button
        title="Отправить заказ"
        onPress={() =>
          Share.share({
            message: `Заказ: ${entree}, ${sideDish}, ${accompaniment}`,
          })
        }
      />
    </View>
  )
}

export default function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator initialRouteName="Entree">
        <Stack.Screen
          name="Entree"
          component={EntreeScreen}
          options={{ title: 'Lunch Tray', headerBackVisible: false }}
        />
        <Stack.Screen name="SideDish" component={SideDishScreen} options={{ title: 'Гарнир' }} />
        <Stack.Screen
          name="Accompaniment"
          component={AccompanimentScreen}
          options={{ title: 'Напиток' }}
        />
        <Stack.Screen name="Checkout" component={CheckoutScreen} options={{ title: 'Чек' }} />
      </Stack.Navigator>
      <StatusBar style="auto" />
    </NavigationContainer>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    padding: 40,
  },
  title: { fontSize: 20, marginBottom: 16 },
})
```

## Итог

- Пошаговый поток — это стек экранов.
- Данные накапливаются через `{ ...route.params, новое_поле }`.
- Кнопка «Назад» в заголовке служит отменой шага.
- `popToTop()` возвращает к началу.
