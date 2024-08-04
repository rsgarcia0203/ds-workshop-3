<script setup lang="ts">
import { reactive, ref, onMounted } from 'vue';
import axios from 'axios';
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const proccess = {
    env: {
        API_BASE_URL: import.meta.env.VITE_API_BASE_URL,
        AWS_REGION: import.meta.env.VITE_AWS_REGION,
        AWS_ACCESS_KEY_ID: import.meta.env.VITE_AWS_ACCESS_KEY_ID,
        AWS_SECRET_ACCESS_KEY: import.meta.env.VITE_AWS_SECRET_ACCESS_KEY,
        AWS_SESSION_TOKEN: import.meta.env.VITE_AWS_SESSION_TOKEN,
        AWS_BUCKET_NAME: import.meta.env.VITE_AWS_BUCKET_NAME
    }
};

const sendingImage = ref(false);
const label = ref('Enviar');
const form = reactive({
    cedula: '',
    nombre: '',
    correo: '',
    photo: ''
});

const items = ref([]);

const s3Client = new S3Client({
    region: proccess.env.AWS_REGION,
    credentials: {
        accessKeyId: proccess.env.AWS_ACCESS_KEY_ID,
        secretAccessKey: proccess.env.AWS_SECRET_ACCESS_KEY,
        sessionToken: proccess.env.AWS_SESSION_TOKEN
    }
});

const bucketName = proccess.env.AWS_BUCKET_NAME;

const tableName = 'sd-workshop3';

const handleFileUpload = async (event) => {
    const image = event.target.files[0];

    if (!image) {
        añert('Por favor seleccione una imagen');
        return;
    }

    if (!image) {
        alert('Por favor seleccione una imagen válida');
        return;
    }

    console.log('Image:', image.name);

    const body = (await image.arrayBuffer()) as Buffer;
    const key = `${Date.now()}-${image.name}`;

    try {
        await s3Client.send(
            new PutObjectCommand({
                Bucket: bucketName,
                Key: key,
                Body: body,
                ContentType: image.type
            })
        );
    } catch (error) {
        console.error('Error:', error);
        return;
    }

    try {
        const getObject = new GetObjectCommand({
            Bucket: bucketName,
            Key: key,
            ACL: 'public-read'
        });

        const url = await getSignedUrl(s3Client, getObject, { expiresIn: 3600 });

        console.log(url);
        form.photo = url;
    } catch (error) {
        console.error('Error:', error);
        return;
    }
};

const submitForm = async () => {
    if (sendingImage.value) {
        alert('Espere un momento mientras se carga la imagen');
        return;
    }
    const data = {
        TableName: tableName,
        Item: {
            nombre: `${form.nombre} ${form.apellido}`,
            cedula: form.cedula,
            correo: form.correo,
            foto: form.photo
        }
    };

    const response = await axios.post(`${proccess.env.API_BASE_URL}/saveResource`, data, {
        headers: {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*'
        }
    });

    console.log('Success:', response.data);

    loadData();
};

const loadData = async () => {
    try {
        const response = await axios.get(
            `${proccess.env.API_BASE_URL}/saveResource?TableName=${tableName}`,
            {
                headers: {
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                }
            }
        );

        items.value = response.data.Items;

        console.log('Success:', response.data);
    } catch (error) {
        console.error('Error:', error);
    }
};

onMounted(() => {
    loadData();
});
</script>
<template>
    <div>
        <h2>Formulario de Inscripción</h2>
        <form @submit.prevent="submitForm">
            <div>
                <label for="firstName">Nombre:</label>
                <input type="text" id="firstName" v-model="form.nombre" required />
            </div>
            <div>
                <label for="lastName">Apellido:</label>
                <input type="text" id="lastName" v-model="form.apellido" required />
            </div>
            <div>
                <label for="idNumber">Número de Cédula:</label>
                <input type="text" id="idNumber" v-model="form.cedula" required />
            </div>
            <div>
                <label for="email">Correo:</label>
                <input type="email" id="email" v-model="form.correo" required />
            </div>
            <div>
                <label for="photo">Foto:</label>
                <input type="file" id="photo" @change="handleFileUpload" required />
            </div>
            <button type="submit">{{ label }}</button>
        </form>

        <h2>Participantes</h2>
        <ul>
            <li v-if="items.length == 0">
                <p>No hay participantes registrados</p>
            </li>
            <li v-else v-for="item in items" :key="item.id">
                <img :src="item.foto" alt="Foto" width="124" height="124" />
                <p>{{ item.nombre }}</p>
                <p>{{ item.cedula }}</p>
                <p>{{ item.correo }}</p>
            </li>
        </ul>

        <h2>Galeria de Imagenes</h2>
        <ul>
            <li v-if="items.length == 0">
                <p>No hay imagenes registradas</p>
            </li>
            <li v-else v-for="item in items" :key="item.id">
                <img :src="item.foto" alt="Foto" width="124" height="124" />
            </li>
        </ul>
    </div>
</template>
