<script>
    import { onMount } from 'svelte';
    import { goto } from '$app/navigation';
    import { HfInference } from "@huggingface/inference";
    import { auth } from "../../stores/auth";
    import { writable } from 'svelte/store';
    import { getFirestore, doc, getDoc } from "firebase/firestore"; // Import Firestore functions

    let isLoading = true;
    let username = '';
    let dietPlan = '';
    let userInput = '';
    let userMetrics = {}; // Store user metrics
    const client = new HfInference("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");
    const db = getFirestore(); // Initialize Firestore

    let isDark = typeof window !== 'undefined' 
        ? localStorage.getItem('theme') === 'dark' 
            || (!localStorage.getItem('theme') && window.matchMedia('(prefers-color-scheme: dark)').matches)
        : false;

    function toggleTheme() {
        isDark = !isDark;
        updateTheme(isDark);
    }

    function updateTheme(dark) {
        if (dark) {
            document.documentElement.classList.add('dark');
            localStorage.setItem('theme', 'dark');
        } else {
            document.documentElement.classList.remove('dark');
            localStorage.setItem('theme', 'light');
        }
    }

    const handleLogout = async () => {
      try {
        await auth.signOut();
        goto('/login');
      } catch (error) {
        console.error('Error logging out:', error);
      }
    };

    async function fetchUserMetrics(userId) {
      try {
        const docRef = doc(db, "users", userId);
        const docSnap = await getDoc(docRef);

        if (docSnap.exists()) {
          userMetrics = docSnap.data();
          console.log("User metrics:", userMetrics);
        } else {
          console.log("No such document!");
        }
      } catch (error) {
        console.error("Error fetching user metrics:", error);
      }
    }

    async function generateDietPlan() {
      try {
        const chatCompletion = await client.chatCompletion({
          model: "microsoft/Phi-3.5-mini-instruct",
          messages: [
            {
              role: "user",
              content: `Generate a diet plan for: ${userInput} with metrics: ${JSON.stringify(userMetrics)}`
            }
          ],
          max_tokens: 500
        });
        dietPlan = chatCompletion.choices[0].message.content;
      } catch (error) {
        console.error('Error generating diet plan:', error);
      }
    }

    onMount(() => {
      updateTheme(isDark);

      setTimeout(() => {
        isLoading = false;
      }, 1000);

      return auth.subscribe((user) => {
        if (!user) {
          console.log('User not authenticated, redirecting to login');
          goto('/login');
        } else {
          username = user.displayName || user.email?.split('@')[0] || 'User';
          console.log('User authenticated:', username);
          fetchUserMetrics(user.uid); // Fetch user metrics on authentication
        }
      });
    });
</script>

{#if isLoading}
<div class="loading-spinner"></div>
{:else}
<div class="min-h-screen" style="background-color: var(--bg-color); color: var(--text-color);">
  <div class="p-4 md:p-6 lg:p-8">
    <nav class="flex flex-wrap gap-4 mb-12 animate-fade-in">
      <button 
        on:click={() => goto('/')}
        class="px-8 py-3 rounded-full text-lg font-medium shadow-sm" 
        style="background-color: var(--card-bg-color); color: var(--card-text-color);"
      >
        Dashboard
      </button>
      <button 
        class="px-8 py-3 rounded-full text-lg font-medium shadow-sm highlight" 
        style="background-color: var(--highlight-bg-color); color: var(--highlight-text-color);"
      >
        Diet Plan
      </button>
      <button 
        on:click={() => goto('/creds')}
        class="px-8 py-3 rounded-full text-lg font-medium shadow-sm" 
        style="background-color: var(--card-bg-color); color: var(--card-text-color);"
      >
        Add New Values
      </button>
      <span class="flex items-center ml-auto mr-4 font-medium" style="color: var(--text-color);">
        Welcome, {username}
      </span>
      <button 
        on:click={handleLogout}
        class="px-8 py-3 rounded-full text-lg font-medium shadow-sm" 
        style="background-color: var(--card-bg-color); color: var(--card-text-color);"
      >
        Logout
      </button>
      <button
        on:click={toggleTheme}
        class="px-8 py-3 rounded-full text-lg font-medium shadow-sm"
        style="background-color: var(--card-bg-color); color: var(--card-text-color);"
      >
        {isDark ? 'Light Mode' : 'Dark Mode'}
      </button>
    </nav>

    <header class="mb-12 animate-fade-in animation-delay-200">
      <h1 class="text-4xl font-medium mb-2" style="color: var(--text-color);">Diet Plan</h1>
      <h2 class="text-4xl font-normal" style="color: var(--text-color);">Personalized Nutrition</h2>
    </header>

    <div class="flex flex-col gap-8">
      <input 
        type="text" 
        placeholder="Enter your dietary preferences or goals" 
        bind:value={userInput} 
        class="p-4 rounded-lg shadow-sm" 
        style="background-color: var(--input-bg-color); color: var(--input-text-color);"
      />
      <button 
        on:click={generateDietPlan}
        class="px-8 py-3 rounded-full text-lg font-medium shadow-sm" 
        style="background-color: var(--button-bg-color); color: var(--button-text-color);"
      >
        Generate Diet Plan
      </button>
      {#if dietPlan}
      <div class="p-8 rounded-3xl shadow-lg" style="background-color: var(--card-bg-color); color: var(--card-text-color);">
        <h3 class="text-2xl font-medium mb-4">Your Diet Plan</h3>
        <p>{dietPlan}</p>
      </div>
      {/if}
    </div>
  </div>
</div>
{/if}

<style>
/* Loading Spinner */
.loading-spinner {
  border: 4px solid #f3f3f3;
  border-top: 4px solid #4867AA;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  animation: spin 1s linear infinite;
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

@keyframes spin {
  0% { transform: translate(-50%, -50%) rotate(0deg); }
  100% { transform: translate(-50%, -50%) rotate(360deg); }
}

/* Fade In Animation */
@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

/* Animation Classes */
.animate-fade-in {
  opacity: 0;
  animation: fadeIn 0.6s ease-out forwards;
}

/* Highlight style for the active button */
.highlight {
  border: 2px solid var(--highlight-color);
  background-color: var(--highlight-bg-color);
  color: var(--highlight-text-color);
}
</style>