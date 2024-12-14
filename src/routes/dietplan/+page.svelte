<script>
    import { onMount } from 'svelte';
    import { goto } from '$app/navigation';
    import { HfInference } from "@huggingface/inference";
    import { auth } from "../../stores/auth";
    import { writable } from 'svelte/store';
    import { getFirestore, doc, getDoc, setDoc } from "firebase/firestore";

    let isLoading = true;
    let isThinking = false;
    let username = '';
    let dietPlan = '';
    let exercisePlan = '';
    let userInput = '';
    let userData = {};
    const client = new HfInference("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");
    const db = getFirestore();

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

    async function fetchUserMetrics(user) {
      try {
        const userDocRef = doc(db, "userdata", user.email);
        const userDoc = await getDoc(userDocRef);

        if (!userDoc.exists() || !userDoc.data().cred) {
          goto('/creds');
        } else {
          const data = userDoc.data();
          if (data.metrics) {
            userData = data.metrics;
          } else {
            const cred = data.cred;
            const height = parseFloat(cred.height) / 100;
            const weight = parseFloat(cred.weight);
            const impedance = parseFloat(cred.impedance);
            const age = parseInt(cred.age);
            const gender = cred.gender;

            const bodyFatPercentage = gender === 'male' 
              ? calculateBodyFatPercentageMale(weight, impedance)
              : calculateBodyFatPercentageFemale(weight, impedance);

            userData = {
              bmi: calculateBMI(weight, height),
              bodyFatPercentage: bodyFatPercentage,
              ffm: calculateFFM(weight, bodyFatPercentage),
              tbw: calculateTBW(weight, height, impedance, gender),
              bmr: calculateBMR(weight, height, age, gender)
            };

            await setDoc(userDocRef, { metrics: userData }, { merge: true });
          }
        }
      } catch (error) {
        console.error("Error checking user credentials:", error);
        goto('/creds');
      }
    }

    async function generateDietPlan() {
      isThinking = true;
      try {
        const chatCompletion = await client.chatCompletion({
          model: "microsoft/Phi-3.5-mini-instruct",
          messages: [
            {
              role: "system",
              content: `
                You are a fitness and nutrition assistant. Analyze health metrics and create personalized plans. Follow this structure:

                1. Display Metrics:
                BMI, Body Fat %, FFM, Weight, Height, Activity Level
                Format: [Metric]: [Value] - [Category]

                2. Create Plans Based On:
                - Goals (loss/gain/maintain)
                - Dietary preferences
                - Health restrictions
                - Time available
                - Equipment access

                Diet Instructions:
                - Show daily calories
                - Macro splits
                - Meal suggestions
                - Timing
                - Brief explanations
              `
            },
            {
              role: "user",
              content: `Generate a diet plan for: ${userInput} with metrics: ${JSON.stringify(userData)}`
            }
          ],
          max_tokens: 1000
        });
        dietPlan = chatCompletion.choices[0].message.content;
      } catch (error) {
        console.error('Error generating diet plan:', error);
      } finally {
        isThinking = false;
      }
    }

    async function generateExercisePlan() {
      isThinking = true;
      try {
        const chatCompletion = await client.chatCompletion({
          model: "microsoft/Phi-3.5-mini-instruct",
          messages: [
            {
              role: "system",
              content: `
                You are a fitness and nutrition assistant. Create an exercise plan based on:

                Exercise Format:
                - Weekly schedule
                - Sets/reps/duration
                - Progressive overload
                - Form cues
                - Alternatives

                Safety:
                Start gradually. Suggest medical clearance when needed. Not medical advice.

                Style:
                Clear, direct, scientific. Use metric/imperial. Focus on practical advice.
              `
            },
            {
              role: "user",
              content: `Generate an exercise plan for: ${userInput} with metrics: ${JSON.stringify(userData)}`
            }
          ],
          max_tokens: 1000
        });
        exercisePlan = chatCompletion.choices[0].message.content;
      } catch (error) {
        console.error('Error generating exercise plan:', error);
      } finally {
        isThinking = false;
      }
    }

    onMount(() => {
      updateTheme(isDark);

      setTimeout(() => {
        isLoading = false;
      }, 1000);

      return auth.subscribe(async (user) => {
        if (!user) {
          goto('/login');
        } else {
          username = user.displayName || user.email?.split('@')[0] || 'User';
          await fetchUserMetrics(user);
        }
      });
    });

    function calculateBMI(weight, height) {
      return weight / (height * height);
    }

    function calculateBodyFatPercentageMale(weight, impedance) {
      // Add your formula here
      return 0; // Placeholder
    }

    function calculateBodyFatPercentageFemale(weight, impedance) {
      // Add your formula here
      return 0; // Placeholder
    }

    function calculateFFM(weight, bodyFatPercentage) {
      return weight * (1 - bodyFatPercentage / 100);
    }

    function calculateTBW(weight, height, impedance, gender) {
      // Add your formula here
      return 0; // Placeholder
    }

    function calculateBMR(weight, height, age, gender) {
      // Add your formula here
      return 0; // Placeholder
    }
</script>

{#if isLoading}
<div class="loading-spinner"></div>
{:else}
<div class="min-h-screen bg-gray-100 dark:bg-gray-900 text-gray-800 dark:text-gray-200">
  <div class="p-6 md:p-8 lg:p-10">
    <nav class="flex flex-wrap gap-4 mb-12 animate-fade-in">
      <button 
        on:click={() => goto('/')}
        class="px-8 py-3 rounded-full text-lg font-medium shadow-md transition-transform transform hover:scale-105" 
        style="background-color: var(--card-bg-color); color: var(--card-text-color);"
      >
        Dashboard
      </button>
      <button 
        class="px-8 py-3 rounded-full text-lg font-medium shadow-md highlight transition-transform transform hover:scale-105" 
        style="background-color: var(--highlight-bg-color); color: var(--highlight-text-color);"
      >
        Diet Plan
      </button>
      <button 
        on:click={() => goto('/creds')}
        class="px-8 py-3 rounded-full text-lg font-medium shadow-md transition-transform transform hover:scale-105" 
        style="background-color: var(--card-bg-color); color: var(--card-text-color);"
      >
        Add New Values
      </button>
      <span class="flex items-center ml-auto mr-4 font-medium">
        Welcome, {username}
      </span>
      <button 
        on:click={handleLogout}
        class="px-8 py-3 rounded-full text-lg font-medium shadow-md transition-transform transform hover:scale-105" 
        style="background-color: var(--card-bg-color); color: var(--card-text-color);"
      >
        Logout
      </button>
      <button
        on:click={toggleTheme}
        class="px-8 py-3 rounded-full text-lg font-medium shadow-md transition-transform transform hover:scale-105"
        style="background-color: var(--card-bg-color); color: var(--card-text-color);"
      >
        {isDark ? 'Light Mode' : 'Dark Mode'}
      </button>
    </nav>

    <header class="mb-12 animate-fade-in animation-delay-200">
      <h1 class="text-4xl font-medium mb-2">Diet Plan</h1>
      <h2 class="text-2xl font-normal">Personalized Nutrition</h2>
    </header>

    <div class="flex flex-col gap-8">
      <input 
        type="text" 
        placeholder="Enter your dietary preferences or goals" 
        bind:value={userInput} 
        class="p-4 rounded-lg shadow-md transition-shadow focus:shadow-lg" 
        style="background-color: var(--input-bg-color); color: var(--input-text-color);"
      />
      <div class="flex gap-4">
        <button 
          on:click={generateDietPlan}
          class="px-8 py-3 rounded-full text-lg font-medium shadow-md transition-transform transform hover:scale-105" 
          style="background-color: var(--button-bg-color); color: var(--button-text-color);"
        >
          Generate Diet Plan
        </button>
        <button 
          on:click={generateExercisePlan}
          class="px-8 py-3 rounded-full text-lg font-medium shadow-md transition-transform transform hover:scale-105" 
          style="background-color: var(--button-bg-color); color: var(--button-text-color);"
        >
          Generate Exercise Plan
        </button>
      </div>
      {#if isThinking}
      <div class="thinking-spinner">Thinking...</div>
      {/if}
      <div class="flex flex-wrap gap-8">
        {#if dietPlan}
        <div class="flex-1 p-8 rounded-3xl shadow-lg bg-white dark:bg-gray-800">
          <h3 class="text-2xl font-medium mb-4">Your Diet Plan</h3>
          <div class="space-y-4">
            {#each dietPlan.split('\n').filter(line => line.trim() !== '') as line}
              <p class="leading-relaxed">{line}</p>
            {/each}
          </div>
        </div>
        {/if}
        {#if exercisePlan}
        <div class="flex-1 p-8 rounded-3xl shadow-lg bg-white dark:bg-gray-800">
          <h3 class="text-2xl font-medium mb-4">Your Exercise Plan</h3>
          <div class="space-y-4">
            {#each exercisePlan.split('\n').filter(line => line.trim() !== '') as line}
              <p class="leading-relaxed">{line}</p>
            {/each}
          </div>
        </div>
        {/if}
      </div>
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

.thinking-spinner {
  font-size: 1.2em;
  color: #666;
  text-align: center;
  margin-top: 20px;
  animation: pulse 1.5s infinite;
}

@keyframes spin {
  0% { transform: translate(-50%, -50%) rotate(0deg); }
  100% { transform: translate(-50%, -50%) rotate(360deg); }
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
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