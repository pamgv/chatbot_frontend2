<template>
  <div class="w-full">
    <div class="form-header">
      <h1>Create Account</h1>
      <p>Register to start using the application</p>
      <p class="privacy-warning">
  <strong>Privacy Notice:</strong> Do not use real names, institutional emails, or any personal information. 
  Only use a simple, anonymous username. Avoid using anything containing “nmsu”.
  <br>Usernames must be <strong>lowercase only</strong>. Uppercase letters are not allowed.
</p>


    </div>

    <form @submit.prevent="handleRegister" class="form">
      <div class="form-group">
        <label for="username">Username</label>
        <input
  type="text"
  id="username"
  v-model="form.username"
  required
  :disabled="loading"
  placeholder="Choose a username"
  minlength="3"
  @input="normalizeUsername"
/>

      </div>

      <div class="password-grid">
          <div class="form-group">
            <label for="password">Password</label>
            <input
              type="password"
              id="password"
              v-model="form.password"
              required
              :disabled="loading"
              placeholder="Create a secure password"
              minlength="6"
            />
          </div>
    
          <div class="form-group">
            <label for="confirmPassword">Confirm Password</label>
            <input
              type="password"
              id="confirmPassword"
              v-model="form.confirmPassword"
              required
              :disabled="loading"
              placeholder="Confirm your password"
            />
          </div>
      </div>


      <button type="submit" :disabled="loading" class="submit-button">
        <span v-if="loading">Creating account...</span>
        <span v-else>Create Account</span>
      </button>
    </form>

    <div class="form-footer">
      <p>
        Already have an account?
        <router-link to="/login" class="link">
          Log in here
        </router-link>
      </p>
    </div>
  </div>
</template>

<script>
import { ref } from 'vue';
import { useRouter } from 'vue-router';
import { useAuthStore } from '../../store/auth';
import { useToast } from 'vue-toastification';

export default {
  name: 'RegisterView',
  setup() {
    const router = useRouter();
    const authStore = useAuthStore();
    const toast = useToast();
    
    const form = ref({
      username: '',
      password: '',
      confirmPassword: ''
    });
    
    const loading = ref(false);

    // 🔤 Convertir automáticamente a minúsculas
    const normalizeUsername = () => {
      form.value.username = form.value.username.toLowerCase();
    };

    const validateForm = () => {
      normalizeUsername();

      if (!form.value.username || !form.value.password || !form.value.confirmPassword) {
        toast.error('Please fill in all fields');
        return false;
      }

      if (form.value.username.length < 3) {
        toast.error('The username must have at least 3 characters');
        return false;
      }

      // ⛔ Bloquear emails
      if (/@/.test(form.value.username) || /\./.test(form.value.username)) {
        toast.error("Do not use emails or personal information in the username.");
        return false;
      }

      // ⛔ Bloquear 'nmsu'
      if (form.value.username.toLowerCase().includes("nmsu")) {
        toast.error("Usernames cannot contain 'nmsu'.");
        return false;
      }

      // ⛔ No espacios
      if (form.value.username.includes(" ")) {
        toast.error("The username cannot contain spaces or full names.");
        return false;
      }

      // ⛔ SOLO minúsculas permitidas (opcional números y '_')
      if (!/^[a-z0-9_]+$/.test(form.value.username)) {
        toast.error("Use only lowercase letters (a–z), numbers, or underscores.");
        return false;
      }

      if (form.value.password.length < 6) {
        toast.error('The password must have at least 6 characters');
        return false;
      }

      if (form.value.password !== form.value.confirmPassword) {
        toast.error('The passwords do not match');
        return false;
      }

      return true;
    };

    const handleRegister = async () => {
      normalizeUsername(); // asegurar minúsculas

      if (!validateForm()) return;

      loading.value = true;

      try {
        const result = await authStore.register(form.value.username, form.value.password);
        
        if (result.success) {
          toast.success(result.message);

          form.value = {
            username: '',
            password: '',
            confirmPassword: ''
          };

          setTimeout(() => {
            router.push('/login');
          }, 2000);
        } else {
          toast.error(result.message);
        }
      } catch (error) {
        toast.error('Error creating the account. Please try again.');
      } finally {
        loading.value = false;
      }
    };

    return {
      form,
      loading,
      handleRegister,
      normalizeUsername
    };
  }
};
</script>



<style scoped src="../../styles/formsAuth.css" />



