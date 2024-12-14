<script lang="ts">
    import { onMount, onDestroy } from "svelte";
    import { page } from "$app/stores";
    import { fade, fly } from "svelte/transition";
    import { goto } from "$app/navigation";
    import { initFirebase } from "$lib/client/firebase";
    import { doc, updateDoc } from "firebase/firestore";
    const {auth, db } = initFirebase();

    const googleAuthUrl = $page.data.authUrl;

    // Initialize isDark based on localStorage or system preference
    let isDark = typeof window !== 'undefined' 
        ? localStorage.getItem('theme') === 'dark' 
            || (!localStorage.getItem('theme') && window.matchMedia('(prefers-color-scheme: dark)').matches)
        : false;

    let mouseX = 0;
    let mouseY = 0;
    let leftContainer: HTMLElement;
    let currentImageIndex = 0;
    let intervalId: ReturnType<typeof setInterval>;
    let isLoaded = false;
    let initialLoad = true;

    const backgroundImages = [
        "/images/11.jpg",
        "/images/12.jpg",
        "/images/13.jpg",
        "/images/14.jpg",
        "/images/15.jpg"
    ];

    let currentUser: any = null;
    
    function toggleTheme() {
        isDark = !isDark;
        updateTheme(isDark);
    }

    function updateTheme(dark: boolean) {
        if (dark) {
            document.documentElement.classList.add('dark');
            localStorage.setItem('theme', 'dark');
        } else {
            document.documentElement.classList.remove('dark');
            localStorage.setItem('theme', 'light');
        }
    }

    function handleSignupRedirect() {
        goto('/signup');
    }

    function handleMouseMove(event: MouseEvent) {
        if (!leftContainer) return;
        const rect = leftContainer.getBoundingClientRect();
        mouseX = ((event.clientX - rect.left) / rect.width) * 100;
        mouseY = ((event.clientY - rect.top) / rect.height) * 100;
    }

    function switchImage() {
        initialLoad = false;
        currentImageIndex = (currentImageIndex + 1) % backgroundImages.length;
    }

    onMount(() => {
        // Apply the initial theme
        updateTheme(isDark);

        setTimeout(() => {
            isLoaded = true;
        }, 100);
        intervalId = setInterval(switchImage, 5000);

        // Subscribe to auth state
        const unsubscribe = auth.onAuthStateChanged((user) => {
            currentUser = user;
            if (!user) {
                goto('/login');
            }
        });

        return () => {
            unsubscribe();
        }
    });

    onDestroy(() => {
        if (intervalId) clearInterval(intervalId);
    });

    async function handleSubmit(event: Event) {
        event.preventDefault();
        
        if (!currentUser) {
            console.error('No user logged in');
            return;
        }

        const formData = new FormData(event.target as HTMLFormElement);
        const credentials = {
            username: formData.get('username'),
            age: Number(formData.get('age')),
            gender: formData.get('gender'),
            weight: Number(formData.get('weight')),
            height: Number(formData.get('height')),
            impedance: Number(formData.get('impedance')),
            updatedAt: new Date()
        };

        try {
            const userDocRef = doc(db, 'userdata', currentUser.email);
            await updateDoc(userDocRef, {
                cred: credentials
            });
            
            // Redirect after successful update
            goto('/');
        } catch (error) {
            console.error('Error updating credentials:', error);
            // Handle error (you might want to show an error message to the user)
        }
    }
</script>

{#if isLoaded}
<div 
    class="min-h-screen w-full relative overflow-hidden" 
    in:fade={{ duration: 300 }}
>
    <!-- Content Grid -->
    <div class="min-h-screen grid lg:grid-cols-2 grid-cols-1">
        <!-- Left Side - Branding -->
        <div
            class="relative lg:min-h-screen min-h-[50vh]"
            bind:this={leftContainer}
            on:mousemove={handleMouseMove}
            role="presentation"
        >
            <!-- Background Container -->
            <div class="absolute inset-0">
                {#each backgroundImages as image, i}
                    <div
                        class="absolute inset-0 transition-all duration-1000"
                        style:opacity={currentImageIndex === i ? "1" : "0"}
                    >
                        <img
                            src={image}
                            alt="Background"
                            class="w-full h-full object-cover transition-transform duration-1000"
                            style="transform: scale({initialLoad || currentImageIndex === i ? '1' : '1.05'})"
                        />
                    </div>
                {/each}
            </div>

            <!-- Content -->
                       <!-- Content -->
                       <div
                       class="relative z-30 flex flex-col justify-center items-center h-full p-4 lg:p-8"
                       in:fly={{ y: 20, duration: 800, delay: 300 }}
                   >
                       <div class="max-w-2xl mx-auto space-y-4 lg:space-y-6">
                           <h1
                               class="text-white/80 text-xl lg:text-2xl text-center hero-text font-[SH AD Grotesk]"
                               in:fly={{ y: 20, duration: 800, delay: 500 }}
                           >
                               Know your health,<br /> monitor your life
                           </h1>
                           <div
                               class="flex justify-between items-center w-full max-w-xl mx-auto mb-8 lg:mb-16 space-x-4 lg:space-x-6"
                               in:fly={{ y: 20, duration: 800, delay: 700 }}
                           >
                               <span class="text-white/80 w-24 lg:w-56 text-lg lg:text-2xl hero-text text-right font-sh-grotesk">
                                   Real-time monitoring
                               </span>
                               <div class="text-[40px] lg:text-[80px] text-white m-0 font-bold animate-pulse">
                                   桜
                               </div>
                               <span class="text-white/80 w-24 lg:w-56 text-lg lg:text-2xl hero-text text-left font-sh-grotesk">
                                   Personalized insights
                               </span>
                           </div>
                       </div>
                   </div>
        </div>

        <!-- Right Side - Login Form -->
        <div
            class="flex items-center justify-center p-6 lg:p-8 bg-white dark:bg-[#0E0E0C]"
            in:fly={{ x: 20, duration: 800 }}
        >
            <div class="w-full max-w-md space-y-6 lg:space-y-8 min-w-[280px] px-4">
                <div class="text-center" in:fly={{ y: 20, duration: 800, delay: 200 }}>
                    <h2 class="text-xl lg:text-2xl font-semibold text-gray-900 dark:text-white">
                        User Details
                    </h2>
                    <p class="mt-2 text-sm lg:text-base text-gray-600 dark:text-gray-400">
                        Please enter your credentials
                    </p>
                </div>

                <div class="space-y-4" in:fly={{ y: 20, duration: 800, delay: 400 }}>
                    <!-- Social Logins -->
                    <!-- <a href={googleAuthUrl}>
                        <button class="w-full flex items-center justify-center gap-3 bg-white hover:bg-gray-50 text-gray-700 p-2.5 lg:p-3 rounded-lg transition-all duration-300 shadow-sm dark:bg-[#0E0E0C] dark:border dark:border-gray-600 dark:hover:bg-gray-700 dark:text-white text-sm lg:text-base">
                            <span class="flex items-center justify-center gap-3">
                                <img src="/icons/google.svg" alt="Google" class="w-5 h-5 dark:invert" />
                                Login with Google
                            </span>
                        </button>
                    </a> -->

                    <div class="relative py-2 lg:py-4">
                        <div class="absolute inset-0 flex items-center">
                            <div class="w-full border-t border-gray-300"></div>
                        </div>
                        <div class="relative flex justify-center text-sm">
                            <span class="px-2 bg-white dark:bg-[#0E0E0C] text-gray-500">
                                Or continue with
                            </span>
                        </div>
                    </div>

                    <!-- Email Form -->
                    <!-- ... existing code ... -->

<!-- Form -->
<form 
    class="space-y-4 lg:space-y-6" 
    on:submit|preventDefault={handleSubmit}
>
    <div>
        <label for="username" class="block text-sm font-medium text-gray-700 dark:text-gray-300">
            Username
        </label>
        <input
            type="text"
            id="username"
            name="username"
            class="mt-1 block w-full px-3 py-2 bg-white border border-gray-300 rounded-lg shadow-sm focus:outline-none focus:ring-2 focus:ring-gray-400 focus:border-transparent transition-all duration-200 dark:bg-[#0E0E0C] dark:border dark:border-gray-600 dark:text-white dark:focus:ring-gray-500"
            placeholder="Enter your username"
            required
        />
    </div>

    <div>
        <label for="age" class="block text-sm font-medium text-gray-700 dark:text-gray-300">
            Age
        </label>
        <input
            type="number"
            id="age"
            name="age"
            min="0"
            max="120"
            class="mt-1 block w-full px-3 py-2 bg-white border border-gray-300 rounded-lg shadow-sm focus:outline-none focus:ring-2 focus:ring-gray-400 focus:border-transparent transition-all duration-200 dark:bg-[#0E0E0C] dark:border dark:border-gray-600 dark:text-white dark:focus:ring-gray-500"
            placeholder="Enter your age"
            required
        />
    </div>

    <div>
        <label for="gender" class="block text-sm font-medium text-gray-700 dark:text-gray-300">
            Gender
        </label>
        <select
            id="gender"
            name="gender"
            class="mt-1 block w-full px-3 py-2 bg-white border border-gray-300 rounded-lg shadow-sm focus:outline-none focus:ring-2 focus:ring-gray-400 focus:border-transparent transition-all duration-200 dark:bg-[#0E0E0C] dark:border dark:border-gray-600 dark:text-white dark:focus:ring-gray-500"
            required
        >
            <option value="">Select gender</option>
            <option value="male">Male</option>
            <option value="female">Female</option>
            <option value="other">Other</option>
        </select>
    </div>

    <div>
        <label for="weight" class="block text-sm font-medium text-gray-700 dark:text-gray-300">
            Weight (kg)
        </label>
        <input
            type="number"
            step="0.1"
            id="weight"
            name="weight"
            min="0"
            class="mt-1 block w-full px-3 py-2 bg-white border border-gray-300 rounded-lg shadow-sm focus:outline-none focus:ring-2 focus:ring-gray-400 focus:border-transparent transition-all duration-200 dark:bg-[#0E0E0C] dark:border dark:border-gray-600 dark:text-white dark:focus:ring-gray-500"
            placeholder="Enter your weight"
            required
        />
    </div>

    <div>
        <label for="height" class="block text-sm font-medium text-gray-700 dark:text-gray-300">
            Height (cm)
        </label>
        <input
            type="number"
            id="height"
            name="height"
            min="0"
            class="mt-1 block w-full px-3 py-2 bg-white border border-gray-300 rounded-lg shadow-sm focus:outline-none focus:ring-2 focus:ring-gray-400 focus:border-transparent transition-all duration-200 dark:bg-[#0E0E0C] dark:border dark:border-gray-600 dark:text-white dark:focus:ring-gray-500"
            placeholder="Enter your height"
            required
        />
    </div>

    <div>
        <label for="impedance" class="block text-sm font-medium text-gray-700 dark:text-gray-300">
            Impedance
        </label>
        <input
            type="number"
            step="0.0001"
            id="impedance"
            name="impedance"
            min="0"
            class="mt-1 block w-full px-3 py-2 bg-white border border-gray-300 rounded-lg shadow-sm focus:outline-none focus:ring-2 focus:ring-gray-400 focus:border-transparent transition-all duration-200 dark:bg-[#0E0E0C] dark:border dark:border-gray-600 dark:text-white dark:focus:ring-gray-500"
            placeholder="Enter impedance value"
            required
        />
    </div>

    <button
        type="submit"
        class="w-full flex justify-center py-2 px-4 border border-transparent rounded-lg shadow-sm text-sm font-medium text-white bg-black hover:bg-gray-800 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-gray-500 transition-colors duration-300 dark:bg-white dark:text-black dark:hover:bg-gray-200"
    >
        Submit
    </button>
</form>

<!-- ... rest of the existing code ... -->
                    <div class="text-center pt-2">
                        <p class="text-sm text-gray-600 dark:text-gray-400">
                            Don't have an account?{' '}
                            <button
                                on:click={handleSignupRedirect}
                                class="text-black dark:text-white hover:underline font-medium"
                            >
                                Sign up
                            </button>
                        </p>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <!-- Theme Toggle -->
    <button
        class="absolute top-4 right-4 z-50 p-2 rounded-lg bg-gray-100 dark:bg-[#0E0E0C] dark:border dark:border-gray-900 text-gray-900 dark:text-white shadow-lg"
        on:click={toggleTheme}
        in:fade={{ duration: 800, delay: 1000 }}
    >
        {#if isDark}
            <img src="/icons/moon.svg" alt="Dark mode" class="w-5 h-5" />
        {:else}
            <img src="/icons/sun.svg" alt="Light mode" class="w-5 h-5" />
        {/if}
    </button>
</div>
{:else}
<div class="min-h-screen w-full bg-black"></div>
{/if}

<style>
    :global(.dark) {
        color-scheme: dark;
    }

    .hero-text {
        text-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
    }

    @keyframes pulse {
        0% { opacity: 0.8; }
        50% { opacity: 1; }
        100% { opacity: 0.8; }
    }

    .animate-pulse {
        animation: pulse 3s infinite;
    }
</style> 