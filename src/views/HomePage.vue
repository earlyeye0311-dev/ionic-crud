<template>
  <ion-page>
    <ion-header>
      <ion-toolbar color="primary">
        <ion-title class="ion-text-center">
          Student Manager
        </ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content class="ion-padding">
      
      <!-- TEST CONNECTION BUTTON -->
      <ion-button expand="block" fill="outline" color="primary" @click="testConnection" style="margin-bottom: 10px;">
        <ion-icon :icon="pulseOutline" slot="start"></ion-icon>
        Test Database
      </ion-button>
      
      <ion-text v-if="connectionStatus" :color="connectionColor" style="display: block; text-align: center; margin-bottom: 20px;">
        {{ connectionStatus }}
      </ion-text>
      
      <!-- FORM SECTION -->
      <ion-card>
        <ion-card-header>
          <ion-card-title>{{ isEditing ? 'Update Student' : 'Add Student' }}</ion-card-title>
        </ion-card-header>
        <ion-card-content>
          <ion-item>
            <ion-label position="floating">Name</ion-label>
            <ion-input v-model="student.name"></ion-input>
          </ion-item>

          <ion-item>
            <ion-label position="floating">Email</ion-label>
            <ion-input v-model="student.email" type="email"></ion-input>
          </ion-item>

          <ion-item>
            <ion-label position="floating">Course</ion-label>
            <ion-input v-model="student.course"></ion-input>
          </ion-item>

          <ion-button expand="block" style="margin-top: 15px;" @click="isEditing ? updateStudent() : addStudent()">
            {{ isEditing ? 'Update' : 'Add' }} Student
          </ion-button>

          <ion-button v-if="isEditing" expand="block" fill="clear" color="medium" @click="cancelEdit">
            Cancel
          </ion-button>
        </ion-card-content>
      </ion-card>

      <!-- STUDENT LIST -->
      <h2 style="margin: 20px 0 10px 0; font-size: 1.1rem; font-weight: 600;">Student List ({{ students.length }})</h2>

      <ion-list>
        <ion-item-sliding v-for="s in students" :key="s.key">
          <ion-item>
            <div slot="start" style="width: 40px; height: 40px; border-radius: 50%; background: linear-gradient(135deg, #6c5ce7, #00cec9); display: flex; align-items: center; justify-content: center; color: white; font-weight: bold; margin-right: 12px;">
              {{ getInitial(s.name) }}
            </div>
            <ion-label>
              <h2 style="font-weight: 600; margin-bottom: 4px;">{{ s.name || 'No Name' }}</h2>
              <p style="font-size: 0.85rem; color: #666;">{{ s.email }}</p>
              <p style="font-size: 0.8rem; color: #999;">{{ s.course }}</p>
            </ion-label>
          </ion-item>
          <ion-item-options side="end">
            <ion-item-option color="primary" @click="editStudent(s)">Edit</ion-item-option>
           <ion-item-option color="danger" @click="deleteStudent(s.key!)">Delete</ion-item-option>
          </ion-item-options>
        </ion-item-sliding>

        <ion-item v-if="students.length === 0">
          <ion-label class="ion-text-center">No students yet. Add one!</ion-label>
        </ion-item>
      </ion-list>

    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted } from 'vue';
import {
  IonPage,
  IonHeader,
  IonToolbar,
  IonTitle,
  IonContent,
  IonCard,
  IonCardHeader,
  IonCardTitle,
  IonCardContent,
  IonItem,
  IonLabel,
  IonInput,
  IonButton,
  IonText,
  IonList,
  IonItemSliding,
  IonItemOptions,
  IonItemOption,
  IonIcon
} from '@ionic/vue';
import { ref as dbRef, push, onValue, update, remove, get } from 'firebase/database';
import { db } from '../firebase';
import { pulseOutline } from 'ionicons/icons';

interface Student {
  key?: string;
  name: string;
  email: string;
  course: string;
}

const student = reactive<Student>({ name: '', email: '', course: '' });
const students = ref<Student[]>([]);
const isEditing = ref(false);
const editKey = ref('');
const connectionStatus = ref('');
const connectionColor = ref('');

const getInitial = (name: string) => {
  if (name && name.length > 0) {
    return name.charAt(0).toUpperCase();
  }
  return 'S';
};

const testConnection = async () => {
  try {
    connectionStatus.value = 'Testing connection...';
    connectionColor.value = 'primary';
    const testRef = dbRef(db, 'students');
    const snapshot = await get(testRef);
    if (snapshot.exists()) {
      const count = Object.keys(snapshot.val()).length;
      connectionStatus.value = `✅ Connected! Found ${count} students.`;
      connectionColor.value = 'success';
    } else {
      connectionStatus.value = '✅ Connected! Database is empty.';
      connectionColor.value = 'success';
    }
  } catch (error: any) {
    connectionStatus.value = `❌ Failed: ${error.message}`;
    connectionColor.value = 'danger';
  }
};

const addStudent = async () => {
  if (!student.name || !student.email || !student.course) {
    alert('Please fill all fields!');
    return;
  }
  try {
    await push(dbRef(db, 'students'), student);
    resetForm();
    connectionStatus.value = '✅ Student added!';
    connectionColor.value = 'success';
  } catch (error: any) {
    alert('Error: ' + error.message);
  }
};

onMounted(() => {
  const studentsRef = dbRef(db, 'students');
  onValue(studentsRef, (snapshot) => {
    const data = snapshot.val();
    students.value = [];
    if (data) {
      Object.keys(data).forEach((key) => {
        students.value.push({ key, ...data[key] });
      });
    }
  });
});

const updateStudent = async () => {
  if (!student.name || !student.email || !student.course) {
    alert('Please fill all fields!');
    return;
  }
  try {
    await update(dbRef(db, `students/${editKey.value}`), student);
    isEditing.value = false;
    editKey.value = '';
    resetForm();
    connectionStatus.value = '✅ Student updated!';
    connectionColor.value = 'success';
  } catch (error: any) {
    alert('Error: ' + error.message);
  }
};

const deleteStudent = async (key: string) => {
  if (confirm('Are you sure?')) {
    try {
      await remove(dbRef(db, `students/${key}`));
      connectionStatus.value = 'Student deleted!';
      connectionColor.value = 'danger';
    } catch (error: any) {
      alert('Error: ' + error.message);
    }
  }
};

const editStudent = (s: Student) => {
  isEditing.value = true;
  editKey.value = s.key!;
  student.name = s.name;
  student.email = s.email;
  student.course = s.course;
};

const cancelEdit = () => {
  isEditing.value = false;
  editKey.value = '';
  resetForm();
};

const resetForm = () => {
  student.name = '';
  student.email = '';
  student.course = '';
};
</script>

<style scoped>
ion-card {
  border-radius: 20px;
  box-shadow: 0 4px 15px rgba(2, 1, 1, 0.05);
  margin-bottom: 10px;
}

ion-item {
  --border-radius: 8px;
  --background: #4e7b7b;
  --min-height: 50px; 
  margin-bottom: 8px;


}

h2 {
  color: #2d3436;
  padding-left: 5px;
}
</style>