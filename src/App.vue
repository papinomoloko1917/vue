<script setup>
import { ref, computed } from 'vue';

// Функция округления до 2 знаков после запятой
const round = (value) => {
    return Math.round(value * 100) / 100;
};

// Функция вычисления tgφ из cosφ
const calcTgφ = (cosφ) => {
    if (!cosφ || cosφ === 0) return 0;
    // tgφ = sin φ / cos φ, где sin φ = √(1 - cos²φ)
    const sinφ = Math.sqrt(1 - Math.pow(cosφ, 2));
    return round(sinφ / cosφ);
};

// Массив электроприемников
const receivers = ref([
    {
        id: 1,
        name: "Электроприемник №1",
        phase: "",
        quantity: "",
        powerOneReceiver: "",
        demandCoefficient: "",
        cosφ: "",
        lccL1: "",
        lccL2: "",
        lccL3: "",
        loadAdjL1: "",
        loadAdjL2: "",
        loadAdjL3: "",
        quadroPowerN: "",
        normalizedEC: "",
        // Рассчитанные значения
        activePower: "",
        reactivePower: "",
        fullPower: "",
        current: ""
    }
]);

let nextId = ref(2);
let calculationAttempted = ref(false);

// Вычисление активной мощности для приемника (в кВт)
const calcActivePower = (receiver, phase) => {
    const lcc = phase === 'L1' ? receiver.lccL1 : (phase === 'L2' ? receiver.lccL2 : receiver.lccL3);
    // P = Pуст * Кс * cosφ, где Pуст (lcc) уже в кВт
    const value = lcc * receiver.demandCoefficient * receiver.cosφ;
    return round(value || 0);
};

// Вычисление реактивной мощности для приемника (в квар)
const calcReactivePower = (receiver, phase) => {
    const lcc = phase === 'L1' ? receiver.lccL1 : (phase === 'L2' ? receiver.lccL2 : receiver.lccL3);
    const tgφ = calcTgφ(receiver.cosφ);
    // Q = Pуст * Кс * tgφ, где Pуст (lcc) уже в кВт
    const value = lcc * receiver.demandCoefficient * tgφ;
    return round(value || 0);
};

// Вычисление полной мощности для приемника (в кВА)
const calcFullPower = (receiver, phase) => {
    const active = calcActivePower(receiver, phase);
    const reactive = calcReactivePower(receiver, phase);
    return round(Math.sqrt(active ** 2 + reactive ** 2));
};

// Суммарные мощности по всем приемникам
// Суммарная активная мощность (в кВт)
const totalActivePower = computed(() => {
    let sum = 0;
    receivers.value.forEach(r => {
        sum += calcActivePower(r, 'L1') + calcActivePower(r, 'L2') + calcActivePower(r, 'L3');
    });
    return round(sum);
});

// Суммарная реактивная мощность (в квар)
const totalReactivePower = computed(() => {
    let sum = 0;
    receivers.value.forEach(r => {
        sum += calcReactivePower(r, 'L1') + calcReactivePower(r, 'L2') + calcReactivePower(r, 'L3');
    });
    return round(sum);
});

// Суммарная полная мощность (в кВА)
const totalFullPower = computed(() => {
    let sum = 0;
    receivers.value.forEach(r => {
        sum += calcFullPower(r, 'L1') + calcFullPower(r, 'L2') + calcFullPower(r, 'L3');
    });
    return round(sum);
});

// Расчет тока для приемника
const calcCurrent = (receiver) => {
    const totalPower = calcActivePower(receiver, 'L1') + calcActivePower(receiver, 'L2') + calcActivePower(receiver, 'L3');
    const U = 220; // Фазное напряжение в В
    // I = P * 1000 / (U * cosφ), где P в кВт, U в В, I в А
    return round((totalPower * 1000) / (U * (receiver.cosφ || 1)));
};

// Функция расчета
const calculate = () => {
    calculationAttempted.value = true;
    receivers.value.forEach(receiver => {
        // Распределение мощности по фазам в зависимости от выбранной фазы
        if (receiver.phase === 'L1') {
            receiver.lccL1 = round(receiver.quantity * receiver.powerOneReceiver);
            receiver.lccL2 = 0;
            receiver.lccL3 = 0;
        } else if (receiver.phase === 'L2') {
            receiver.lccL1 = 0;
            receiver.lccL2 = round(receiver.quantity * receiver.powerOneReceiver);
            receiver.lccL3 = 0;
        } else if (receiver.phase === 'L3') {
            receiver.lccL1 = 0;
            receiver.lccL2 = 0;
            receiver.lccL3 = round(receiver.quantity * receiver.powerOneReceiver);
        }

        // Расчет нагрузки с учетом коэффициента спроса и tgφ (в квар)
        const tgφ = calcTgφ(receiver.cosφ);
        receiver.loadAdjL1 = round(receiver.lccL1 * receiver.demandCoefficient * tgφ);
        receiver.loadAdjL2 = round(receiver.lccL2 * receiver.demandCoefficient * tgφ);
        receiver.loadAdjL3 = round(receiver.lccL3 * receiver.demandCoefficient * tgφ);

        // Расчет квадрата мощности (в кВт²)
        receiver.quadroPowerN = round(receiver.quantity * Math.pow(receiver.powerOneReceiver, 2));

        // Нормированный коэффициент эффективности
        let totalInstalledPower = receiver.lccL1 + receiver.lccL2 + receiver.lccL3;
        if (receiver.quadroPowerN !== 0) {
            receiver.normalizedEC = round(Math.pow(totalInstalledPower, 2) / receiver.quadroPowerN);
        } else {
            receiver.normalizedEC = 0;
        }

        // Расчет мощностей (активная в кВт, реактивная в квар, полная в кВА)
        receiver.activePower = round(
            calcActivePower(receiver, 'L1') +
            calcActivePower(receiver, 'L2') +
            calcActivePower(receiver, 'L3')
        );

        receiver.reactivePower = round(
            calcReactivePower(receiver, 'L1') +
            calcReactivePower(receiver, 'L2') +
            calcReactivePower(receiver, 'L3')
        );

        receiver.fullPower = round(
            calcFullPower(receiver, 'L1') +
            calcFullPower(receiver, 'L2') +
            calcFullPower(receiver, 'L3')
        );

        receiver.current = calcCurrent(receiver); // Ток в А (амперах)
    });
};

// Функция очистки формы
const clearForm = () => {
    calculationAttempted.value = false;
    receivers.value = [{
        id: 1,
        name: "Электроприемник №1",
        phase: "",
        quantity: "",
        powerOneReceiver: "",
        demandCoefficient: "",
        cosφ: "",
        lccL1: "",
        lccL2: "",
        lccL3: "",
        loadAdjL1: "",
        loadAdjL2: "",
        loadAdjL3: "",
        quadroPowerN: "",
        normalizedEC: "",
        activePower: "",
        reactivePower: "",
        fullPower: "",
        current: ""
    }];
    nextId.value = 2;
};

// Функция добавления нового электроприемника
const addReceiver = () => {
    receivers.value.push({
        id: nextId.value,
        name: `Электроприемник №${nextId.value}`,
        phase: "",
        quantity: "",
        powerOneReceiver: "",
        demandCoefficient: "",
        cosφ: "",
        lccL1: "",
        lccL2: "",
        lccL3: "",
        loadAdjL1: "",
        loadAdjL2: "",
        loadAdjL3: "",
        quadroPowerN: "",
        normalizedEC: "",
        activePower: "",
        reactivePower: "",
        fullPower: "",
        current: ""
    });
    nextId.value++;
};

// Проверка заполнения обязательных полей
const isFieldInvalid = (value) => {
    return calculationAttempted.value && (value === '' || value === null || value === undefined);
};

// Функция удаления электроприемника
const removeReceiver = (id) => {
    if (receivers.value.length > 1) {
        receivers.value = receivers.value.filter(r => r.id !== id);
    }
};

</script>

<template>
    <div>
        <table class="styled-table">
            <thead>
                <tr>
                    <th rowspan="2"><span>№</span></th>
                    <th rowspan="2"><span>Наименование ЭП</span></th>
                    <th rowspan="2" class="vertical-text"><span>Фаза</span></th>
                    <th rowspan="2" class="vertical-text"><span>Кол-во ЭП</span></th>
                    <th rowspan="2" class="vertical-text"><span>P<sub>уст</sub> одного ЭП, кВт</span></th>
                    <th colspan="3"><span>P<sub>уст</sub> однофазных ЭП, включенных на U<sub>ф</sub>,
                            кВт</span></th>
                    <th rowspan="2" class="vertical-text"><span>К<sub>с</sub></span></th>
                    <th rowspan="2" class="vertical-text"><span>cos<sub>φ</sub></span></th>
                    <th rowspan="2" class="vertical-text"><span>tg<sub>φ</sub></span></th>
                    <th colspan="7"><span>Расчетные величины</span></th>
                    <th rowspan="2" class="vertical-text">
                        <span>n<sub>э</sub>=(∑P<sub>н</sub>)<sup>2</sup>/∑np<sub>н</sub><sup>2</sup></span>
                    </th>
                    <th rowspan="2" class="vertical-text">
                        <span>I<sub>р</sub>=К<sub>с</sub>Р<sub>н</sub>/U<sub>cosφ</sub>,А</span>
                    </th>
                    <th colspan="3"><span>Расчетная мощность</span></th>
                    <th rowspan="2"><span>Действия</span></th>
                </tr>
                <tr>
                    <th><span>L1</span></th>
                    <th><span>L2</span></th>
                    <th><span>L3</span></th>
                    <!-- Расчетные величины -->
                    <th class="vertical-text"><span>К<sub>с</sub>Р<sub>н</sub>(L1)</span></th>
                    <th class="vertical-text"><span>К<sub>с</sub>Р<sub>н</sub>(L2)</span></th>
                    <th class="vertical-text"><span>К<sub>с</sub>Р<sub>н</sub>(L3)</span></th>
                    <th class="vertical-text"><span>К<sub>с</sub>Р<sub>н</sub>tg<sub>φ</sub>(L1)</span></th>
                    <th class="vertical-text"><span>К<sub>с</sub>Р<sub>н</sub>tg<sub>φ</sub>(L2)</span></th>
                    <th class="vertical-text"><span>К<sub>с</sub>Р<sub>н</sub>tg<sub>φ</sub>(L3)</span></th>
                    <th class="vertical-text"><span>np<sub>н</sub><sup>2</sup></span></th>
                    <!--  -->
                    <th class="vertical-text"><span>Активная,кВт</span></th>
                    <th class="vertical-text"><span>Реактивная,квар</span></th>
                    <th class="vertical-text"><span>Полная,кВА</span></th>
                </tr>
            </thead>
            <tbody>
                <tr v-for="(receiver, index) in receivers" :key="receiver.id">
                    <td>{{ index + 1 }}</td>
                    <td><input v-model="receiver.name" type="text" class="name-input"></td>
                    <!-- Выбор фазы -->
                    <td><select v-model="receiver.phase" :class="{ 'invalid-field': isFieldInvalid(receiver.phase) }"
                            required>
                            <option value="" disabled>Выбрать</option>
                            <option value="L1">L1</option>
                            <option value="L2">L2</option>
                            <option value="L3">L3</option>
                        </select></td>
                    <!-- Кол-во ЭП -->
                    <td><input v-model.number="receiver.quantity"
                            :class="{ 'invalid-field': isFieldInvalid(receiver.quantity) }" type="number" step="1"
                            min="1" max="1000" required>
                    </td>
                    <!-- Установленная мощность одного электроприемника -->
                    <td><input v-model.number="receiver.powerOneReceiver"
                            :class="{ 'invalid-field': isFieldInvalid(receiver.powerOneReceiver) }" type="number"
                            step="1" min="1" max="1000" required>
                    </td>
                    <!-- Установленная мощность однофазных ЭП включенных на фазное напряжение -->
                    <td>{{ round(receiver.lccL1) || '' }}</td>
                    <td>{{ round(receiver.lccL2) || '' }}</td>
                    <td>{{ round(receiver.lccL3) || '' }}</td>
                    <!-- Коэффициент спроса -->
                    <td><input v-model.number="receiver.demandCoefficient"
                            :class="{ 'invalid-field': isFieldInvalid(receiver.demandCoefficient) }" type="number"
                            step="0.01" min="0.01" max="1" required>
                    </td>
                    <!-- cosφ -->
                    <td><input v-model.number="receiver.cosφ"
                            :class="{ 'invalid-field': isFieldInvalid(receiver.cosφ) }" type="number" step="0.01"
                            min="0.01" max="1" required>
                    </td>
                    <!-- tgφ (автоматический расчет) -->
                    <td>{{ calcTgφ(receiver.cosφ) || '' }}</td>
                    <!-- Расчетные характеристики -->
                    <td>{{ round(receiver.lccL1) || '' }}</td>
                    <td>{{ round(receiver.lccL2) || '' }}</td>
                    <td>{{ round(receiver.lccL3) || '' }}</td>
                    <td>{{ round(receiver.loadAdjL1) || '' }}</td>
                    <td>{{ round(receiver.loadAdjL2) || '' }}</td>
                    <td>{{ round(receiver.loadAdjL3) || '' }}</td>
                    <!-- Квадрат номинально мощности * кол-во ЭП -->
                    <td>{{ round(receiver.quadroPowerN) || '' }}</td>
                    <!-- Нормированный коэффициент эффективности -->
                    <td>{{ round(receiver.normalizedEC) || '' }}</td>
                    <!-- Расчет тока -->
                    <td>{{ receiver.current || '' }}</td>
                    <!-- Активная мощность -->
                    <td>{{ receiver.activePower || '' }}</td>
                    <!-- Реактивная мощность -->
                    <td>{{ receiver.reactivePower || '' }}</td>
                    <!-- Полная мощность -->
                    <td>{{ receiver.fullPower || '' }}</td>
                    <!-- Кнопка удаления -->
                    <td>
                        <button @click="removeReceiver(receiver.id)" class="delete-btn"
                            :disabled="receivers.length === 1" title="Удалить строку">✖</button>
                    </td>
                </tr>
            </tbody>
        </table>
    </div>
    <div class="btn">
        <button @click="calculate">Рассчитать</button>
        <button @click="clearForm">Очистить форму</button>
        <button @click="addReceiver">Добавить ЭП</button>
    </div>
</template>

<style scoped>
.styled-table {
    width: 100%;
    border-collapse: collapse;
    /* Убирает двойные границы между ячейками */
}

.styled-table th,
.styled-table td {
    padding: 10px;
    text-align: center;
    border: 1px solid #bbbbbb;
    /* Легкая граница вокруг каждой ячейки */
}

.styled-table th {
    background-color: #f2f2f2;
    /* Легкий фон для заголовков */
}

.styled-table tr:nth-child(even) {
    background-color: #f9f9f9;
    /* Альтернативная цветовая заливка для строк */
}

.styled-table tr:hover {
    background-color: #eaeaea;
    /* Цвет фона строки при наведении */
}

.vertical-text {
    height: 150px;
    padding: 5px;
    white-space: nowrap;
}

.vertical-text span {
    writing-mode: vertical-rl;
    transform: rotate(180deg);
    display: inline-block;
}

.btn {
    margin-top: 20px;
    display: flex;
    gap: 15px;
}

.btn button {
    padding: 10px 20px;
    cursor: pointer;
    border: 1px solid #ccc;
    background-color: #f2f2f2;
    border-radius: 4px;
    transition: background-color 0.3s;
}

.btn button:hover {
    background-color: #e0e0e0;
}

.invalid-field {
    border: 1px solid red !important;
}

input.invalid-field,
select.invalid-field {
    border-color: red;
}

.delete-btn {
    color: rgb(255, 0, 0);
    border: none;
    border-radius: 4px;
    padding: 5px 10px;
    cursor: pointer;
    font-size: 14px;
    transition: background-color 0.3s;
}

.delete-btn:disabled {
    cursor: not-allowed;
    opacity: 0.6;
}

.name-input {
    width: 100%;
    padding: 5px;
    border: 1px solid #ccc;
    border-radius: 3px;
    box-sizing: border-box;
    font-size: 14px;
}

.name-input:focus {
    outline: none;
    border-color: #4CAF50;
}
</style>