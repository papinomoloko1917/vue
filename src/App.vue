<script setup>
import { ref } from 'vue';
let firstname = ref("");
let lastname = ref("");
let isDone = ref(false);
let done = () => {
    isDone.value = true;
}
let clearName = () => {
    firstname.value = "";
    lastname.value = "";
}

let send = ref(false);
let sendApply = () => {
    send.value = true;
}

</script>

<template>
    <div v-if="!isDone && !send" @click="done">
        <button>Ввести данные пользователя</button>
    </div>
    <div v-else-if="isDone && !send" class="inpt">
        <label for="first-name">Имя</label>
        <input id="first-name" v-model="firstname">
        <label for="last-name">Фамилия</label>
        <input id="last-name" v-model="lastname">
        <button @click="sendApply">Отправить</button>
        <button @click="clearName">Удалить</button>
    </div>
    <div v-else>
        <h2>
            {{ firstname + lastname }}
        </h2>
    </div>
</template>