<script setup>
import { ref, onMounted } from "vue";
import * as api from "./api/post.js";
import AuthForm from "./components/AuthForm.vue";
import AppHeader from "./components/AppHeader.vue";
import PostList from "./components/PostList.vue";
import PostLoader from "./components/PostLoader.vue";
import PostForm from "./components/PostForm.vue"; // Upewnij się, że masz ten komponent

// STAN APLIKACJI
const currentUser = ref(JSON.parse(localStorage.getItem("user")) || null);
const posts = ref([]);
const isLoadingPosts = ref(false);
const postsError = ref(false);

// STAN SIDEBARU
const isSidebarOpen = ref(false);
const selectedPost = ref(null);

// LOGIKA LOGOWANIA / WYLOGOWANIA
const handleLogin = (user) => {
  currentUser.value = user;
  localStorage.setItem("user", JSON.stringify(user));
  fetchPosts();
};

const handleLogout = () => {
  currentUser.value = null;
  localStorage.removeItem("user");
  posts.value = [];
  isSidebarOpen.value = false;
};

// POBIERANIE POSTÓW
const fetchPosts = async () => {
  if (!currentUser.value) return;

  isLoadingPosts.value = true;
  postsError.value = false;

  try {
    posts.value = await api.getPosts(currentUser.value.id);
  } catch (err) {
    postsError.value = true;
  } finally {
    isLoadingPosts.value = false;
  }
};

// SIDEBAR ACTIONS
const openCreateForm = () => {
  selectedPost.value = null;
  isSidebarOpen.value = true;
};

const openPostInSidebar = (post) => {
  if (selectedPost.value?.id === post.id) {
    closeSidebar();
  } else {
    selectedPost.value = post;
    isSidebarOpen.value = true;
  }
};

const closeSidebar = () => {
  isSidebarOpen.value = false;
  selectedPost.value = null;
};

const handlePostSubmit = async (data) => {
  try {
    if (data.id) {
      const updatedPost = await api.updatePost(data);
      const index = posts.value.findIndex((p) => p.id === data.id);
      posts.value[index] = updatedPost;
      selectedPost.value = updatedPost;
    } else {
      const newPost = await api.createPost({
        ...data,
        userId: currentUser.value.id,
      });
      posts.value.push(newPost);
      selectedPost.value = newPost;
    }

  } catch (err) {
    console.error("Błąd zapisu posta");
  }
};

const handleDeletePost = async (postId) => {
  try {
    await api.deletePost(postId);
    posts.value = posts.value.filter((p) => p.id !== postId);
    closeSidebar();
  } catch (err) {
    console.error("Błąd usuwania posta");
  }
};


onMounted(() => {
  if (currentUser.value) {
    fetchPosts();
  }
});
</script>

<template>
  <div id="app">
    <AuthForm v-if="!currentUser" @login-success="handleLogin" />

    <div v-else>
      <AppHeader :user="currentUser" @logout="handleLogout" />

      <main class="section">
        <div class="container">
          <div class="tile is-ancestor">
            <div
              class="tile is-parent"
              :class="isSidebarOpen ? 'is-4-desktop' : ''">
              <PostList
                :posts="posts"
                :is-loading="isLoadingPosts"
                :has-error="postsError"
                :selected-post-id="selectedPost?.id"
                @add-post="openCreateForm"
                @open-post="openPostInSidebar" />
            </div>

            <div v-if="isSidebarOpen" class="tile is-parent is-8-desktop">
              <PostForm
                :post="selectedPost"
                @close="closeSidebar"
                @submitted="handlePostSubmit"
                @delete="handleDeletePost" />
            </div>
          </div>
        </div>
      </main>
    </div>
  </div>
</template>
