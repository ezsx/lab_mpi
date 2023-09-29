### Эта программа MPI (интерфейс передачи сообщений) предназначена для генерации случайных чисел в некоторых рангах, сортировки этих чисел в двух других рангах и, наконец, объединения отсортированных списков в основном ранге.

# Шаг 1. Генерация случайных чисел

Каждый из `rank` от 3 до `size` - 1, генерирует случайное число. Это делается с помощью `Random` класса Java. Сгенерированное число сохраняется в `randNums` массиве по индексу, соответствующему текущему `rank - 3`.

```java
Random rand = new Random();
int[] randNums = new int[size - 3];
if (rank >= 3) {
    randNums[rank - 3] = rand.nextInt(100);
}
```

# Шаг 2. Отправка и получение случайных чисел

`ranks` от 3 до `size - 1` отправляют свои сгенерированные числа `ranks` 1 и 2, используя неблокирующий метод отправки `Isend`.

```java
MPI.COMM_WORLD.Isend(randNums, rank - 3, 1, MPI.INT, 1, 0);
MPI.COMM_WORLD.Isend(randNums, rank - 3, 1, MPI.INT, 2, 0);
```
Затем `ranks` 1 и 2 получают эти числа, используя неблокирующий метод приема.

```java
Request[] requests = new Request[size - 3];
int[] numbers = new int[size - 3];
for (int i = 3; i < size; i++) {
    requests[i - 3] = MPI.COMM_WORLD.Irecv(numbers, i - 3, 1, MPI.INT, i, 0);
}
Request.Waitall(requests);
```
# Шаг 3. Сортировка и отправка отсортированных чисел

Полученные числа в `ranks` 1 и 2 сортируются. `rank` 1 получает первую половину отсортированных чисел, а `rank` 2 — вторую половину. Отсортированные числа затем отправляются обратно в `rank` 0 с помощью `Isend`.

```java
Arrays.sort(numbers);
if (rank == 1) {
    randNums = Arrays.copyOfRange(numbers, 0, numbers.length / 2);
} else {
    randNums = Arrays.copyOfRange(numbers, numbers.length / 2, numbers.length);
}
MPI.COMM_WORLD.Isend(randNums, 0, randNums.length, MPI.INT, 0, 0);

```

# Шаг 4. Объединение отсортированных списков

`rank` 0 проверяет входящие сообщения от `ranks` 1 и 2 с помощью `Iprobe` метода. Если сообщение доступно, оно получает сообщение, используя `Recv` метод.

```java
int[] list1 = null, list2 = null;
Status status;
while (true) {
    status = MPI.COMM_WORLD.Iprobe(1, 0);
    if (status != null) {
        list1 = new int[status.Get_count(MPI.INT)];
        MPI.COMM_WORLD.Recv(list1, 0, list1.length, MPI.INT, 1, 0);
        break;
    }
}
while (true) {
    status = MPI.COMM_WORLD.Iprobe(2, 0);
    if (status != null) {
        list2 = new int[status.Get_count(MPI.INT)];
        MPI.COMM_WORLD.Recv(list2, 0, list2.length, MPI.INT, 2, 0);
        break;
    }
}

```
Полученные числа из `ranks` 1 и 2 затем объединяются в порядке возрастания и таким образом тоже сортируются.

```java
 int[] mergedList = new int[list1.length + list2.length];
            int i = 0, j = 0, k = 0;
            while (i < list1.length && j < list2.length) {
                if (list1[i] <= list2[j]) {
                    mergedList[k++] = list1[i++];
                } else {
                    mergedList[k++] = list2[j++];
                }
            }
            while (i < list1.length) {
                mergedList[k++] = list1[i++];
            }
            while (j < list2.length) {
                mergedList[k++] = list2[j++];
            }
```

# Шаг 5: Вывод результирующего списка

Объединенный список чисел затем выводится `rank` 0.

```java
for (int num : mergedList) {
    System.out.println(num);
}

```

