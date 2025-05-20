<script lang="ts">
	import { writable } from 'svelte/store';
	import { onMount } from 'svelte';
	import { browser, dev } from '$app/environment';
	import { page } from '$app/stores';
	import  logo  from '../logos/S25_Wordmark.png';
	import { base } from '$app/paths';
	import { createClient } from '@supabase/supabase-js';
	import type { SupabaseClient } from '@supabase/supabase-js';


	type TierItem = { src: string; name: string };
	let uploadedImages: { src: string; name: string }[] = [];
	let tiers = writable<{ name: string; color: string; items: TierItem[] }[]>([
		{ name: 'S', color: '#00FF00', items: [] },
		{ name: 'A', color: '#7FFF00', items: [] },
		{ name: 'B', color: '#FFFF00', items: [] },
		{ name: 'C', color: '#FFBF00', items: [] },
		{ name: 'D', color: '#FF8000', items: [] },
		{ name: 'E', color: '#FF4000', items: [] },
		{ name: 'F', color: '#FF0000', items: [] }
	]);

	const isDragging = writable(false);
	let touchStartX: number | null = null;
	let touchStartY: number | null = null;
	let draggedImage: { src: string; name: string } | null = null;
	let draggedSource: string | null = null;
	let draggedIndex: number | null = null;
	let draggedSourceTierIndex: number | null = null;
	let isOverBin = writable(false);

	let tierListName = writable('');
	let isEditing = writable(false);
	let showContent = writable(false);

	// Initialize Supabase client with error handling
	let supabase: SupabaseClient | undefined;

	try {
		const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
		const supabaseKey = import.meta.env.VITE_SUPABASE_ANON_KEY;
		
		if (!supabaseUrl || !supabaseKey) {
			console.error('Supabase credentials not found in environment variables');
		} else {
			supabase = createClient(supabaseUrl, supabaseKey);
		}
	} catch (error) {
		console.error('Error initializing Supabase client:', error);
	}

	// Define imageCategories as a writable store or a regular variable
	let imageCategories: Record<string, { src: string; name: string }[]> = {};

	// Reintroduce selectedCategory and showCategoryImages
	let selectedCategory: string | null = null;
	let showCategoryImages = false;

	// Add loading state for categories
	let loadingCategories = writable(true);

	// Add this with your other store definitions
	const loadingCategoryId = writable<string | null>(null);

	// Add Supabase image fetching with error handling
	async function fetchImagesFromSupabase() {
		loadingCategories.set(true);
		
		if (!supabase) {
			console.error('Supabase client not initialized');
			loadingCategories.set(false);
			return {};
		}

		try {
			// First, get the list of folders (categories)
			const { data: folders, error: foldersError } = await supabase
				.storage
				.from('images')
				.list('tier-list');

			if (foldersError) {
				console.error('Error fetching folders:', foldersError);
				loadingCategories.set(false);
				return {};
			}

			const supabaseImageCategories: Record<string, { src: string; name: string }[]> = {};

			// For each folder, get its contents
			for (const folder of folders) {
				if (folder.name) {  // Skip if not a folder
					const { data: images, error: imagesError } = await supabase
						.storage
						.from('images')
						.list(`tier-list/${folder.name}`);

					if (imagesError) {
						console.error(`Error fetching images from ${folder.name}:`, imagesError);
						continue;
					}

					// Initialize category array
					supabaseImageCategories[folder.name] = [];

					// Process each image in the folder
					for (const image of images) {
						if (image.name) {  // Skip if not a file
							const { data: { publicUrl } } = supabase
								.storage
								.from('images')
								.getPublicUrl(`tier-list/${folder.name}/${image.name}`);

							supabaseImageCategories[folder.name].push({
								src: publicUrl,
								name: image.name.split('.')[0] // Remove file extension
							});
						}
					}
				}
			}

			loadingCategories.set(false);
			return supabaseImageCategories;
		} catch (error) {
			console.error('Error in fetchImagesFromSupabase:', error);
			loadingCategories.set(false);
			return {};
		}
	}

	onMount(async () => {
			try {
				tierListName.set(localStorage.getItem('tierListName') || '');
				uploadedImages = JSON.parse(localStorage.getItem('uploadedImages') || '[]');
				tiers.set(JSON.parse(localStorage.getItem('tiers') || JSON.stringify($tiers)));
				
				// Only fetch from Supabase if client is initialized
				if (supabase) {
					const supabaseImages = await fetchImagesFromSupabase();
					imageCategories = { ...imageCategories, ...supabaseImages };

				}


			} catch (error) {
				console.error('Error in onMount:', error);
			}
	});

	$: if (browser) {
		localStorage.setItem('tierListName', $tierListName);
		localStorage.setItem('uploadedImages', JSON.stringify(uploadedImages));
		localStorage.setItem('tiers', JSON.stringify($tiers));
	}

	function handleFileUpload(event: Event) {
		const files = (event.target as HTMLInputElement).files;
		if (files) {
			for (const file of files) {
				const reader = new FileReader();
				reader.onload = (e) => {
					const imageData = e.target?.result as string;
					const img = new Image();
					img.onload = () => {
						const canvas = document.createElement('canvas');
						const maxSize = 96; // Maximum width or height (changed to 24)
						let width = img.width;
						let height = img.height;

						// Calculate new dimensions while maintaining aspect ratio
						if (width > height) {
							if (width > maxSize) {
								height *= maxSize / width;
								width = maxSize;
							}
						} else {
							if (height > maxSize) {
								width *= maxSize / height;
								height = maxSize;
							}
						}

						canvas.width = width;
						canvas.height = height;
						const ctx = canvas.getContext('2d');
						ctx?.drawImage(img, 0, 0, width, height);

						// Convert to WebP with quality setting
						canvas.toBlob((blob) => {
							if (blob) {
								const newImage = { 
									src: URL.createObjectURL(blob), 
									name: file.name,
									originalSize: file.size,
									optimizedSize: blob.size
								};
								uploadedImages = [...uploadedImages, newImage];
								console.log(`Reduced from ${file.size/1024}KB to ${blob.size/1024}KB`);
							}
						}, 'image/webp', 0.8); // Use WebP with 80% quality
					};
					img.src = imageData;
				};
				reader.readAsDataURL(file);
			}
		}
	}

	function handleDragStart(event: DragEvent, image: TierItem, source: string, index: number, sourceTierIndex?: number) {
		if (event.dataTransfer) {
			event.dataTransfer.effectAllowed = 'move';
			event.dataTransfer.setData('text/plain', JSON.stringify({ image, source, index }));
		}
		isDragging.set(true);
		draggedImage = image;
		draggedSource = source;
		draggedIndex = index;
		draggedSourceTierIndex = sourceTierIndex ?? null;
	}

	function handleTouchStart(event: TouchEvent, image: TierItem, source: string, index: number, sourceTierIndex?: number) {
		const touch = event.touches[0];
		touchStartX = touch.clientX;
		touchStartY = touch.clientY;
		draggedImage = image;
		draggedSource = source;
		draggedIndex = index;
		draggedSourceTierIndex = sourceTierIndex ?? null;
		isDragging.set(true);

		const imgElement = event.target as HTMLImageElement;
		imgElement.setAttribute('dragging', 'true');
		imgElement.classList.add('fixed', 'max-w-24', 'max-h-24', 'object-contain', 'pointer-events-none', 'z-50', 'cursor-grabbing');
		updateDragImagePosition(touch.clientX, touch.clientY, imgElement);
	}

	function handleTouchMove(event: TouchEvent) {


		if (touchStartX !== null && touchStartY !== null) {
			const touch = event.touches[0];
			const dropTarget = document.elementFromPoint(touch.clientX, touch.clientY);
			isOverBin.set(dropTarget?.closest('.delete-bin') !== null);
			
			// Auto-scroll functionality
			const scrollThreshold = 100; // pixels from top/bottom to trigger scroll
			const scrollSpeed = 10; // pixels per scroll
			const viewportHeight = window.innerHeight;
			const touchY = touch.clientY;
			
			// Scroll up if near top of viewport
			if (touchY < scrollThreshold) {
				window.scrollBy(0, -scrollSpeed);
			}
			// Scroll down if near bottom of viewport
			else if (touchY > viewportHeight - scrollThreshold) {
				window.scrollBy(0, scrollSpeed);
			}
			
			const deltaX = touch.clientX - touchStartX;
			const deltaY = touch.clientY - touchStartY;
			if (Math.abs(deltaX) > 10 || Math.abs(deltaY) > 10) {
				event.preventDefault();
				const imgElement = event.target as HTMLImageElement;
				updateDragImagePosition(touch.clientX, touch.clientY, imgElement);
			}

			if (dropTarget?.closest('.tier')) {
				const tierElement = dropTarget.closest('.tier');
				tierElement?.classList.remove('bg-neutral-800');
				tierElement?.classList.add('h-24', 'bg-neutral-600', 'z-50');
			} else {
				// Remove h-32 from all tiers when not dragging over any tier
				document.querySelectorAll('.tier').forEach(tier => {
					tier.classList.remove('h-24', 'bg-neutral-600', 'z-50');
					tier.classList.add('bg-neutral-800');
				});
			}

			if (dropTarget?.closest('.category-images')) {
				const categoryElement = dropTarget.closest('.category-images');
				categoryElement?.classList.add('bg-neutral-600');
			} else {
				document.querySelectorAll('.category-images').forEach(category => {
					category.classList.remove('bg-neutral-600');
				});
			}
		}
	}

	function handleTouchEnd(event: TouchEvent) {
		if (draggedImage && draggedSource !== null && draggedIndex !== null) {
			const touch = event.changedTouches[0];
			const dropTarget = document.elementFromPoint(touch.clientX, touch.clientY);

			

			if (dropTarget?.classList.contains('delete-bin')) {
				handleBinDrop();
			} else if (dropTarget?.closest('.tier')) {
				const tierElement = dropTarget.closest('.tier');
				const tierIndex = parseInt(tierElement?.getAttribute('data-index') || '-1', 10);
				if (browser && 'ontouchstart' in window) {
				tierElement?.classList.remove('bg-neutral-600', 'z-50');
				if (tierIndex >= 0) {
					handleDrop(tierIndex);
				}
			}
			} else if (dropTarget?.closest('.category-images')) {
				// If dragging within the same category, do nothing
				if (draggedSource === 'uploaded' && selectedCategory) {
					imageCategories[selectedCategory] = [...imageCategories[selectedCategory], draggedImage];
					uploadedImages = uploadedImages.filter((_, i) => i !== draggedIndex);
				}
				if (draggedSource === 'tier' && selectedCategory) {
					// Remove from tier
					if (draggedSourceTierIndex !== null) {
						tiers.update(t => {
							const newTiers = [...t];
							newTiers[draggedSourceTierIndex!].items = newTiers[draggedSourceTierIndex!].items.filter((_, i) => i !== draggedIndex);
							return newTiers;
						});
					}
					// Add back to category
					imageCategories[selectedCategory] = [...imageCategories[selectedCategory], draggedImage];
				} else if (draggedSource === 'category' && selectedCategory) {
					// Only remove if it's being dragged to a different position
					const newIndex = Array.from(dropTarget.closest('.category-images')?.querySelectorAll('img') || [])
						.findIndex(img => img === dropTarget);
					if (newIndex !== draggedIndex) {
						imageCategories[selectedCategory] = imageCategories[selectedCategory].filter((_, i) => i !== draggedIndex);
						imageCategories[selectedCategory].splice(newIndex, 0, draggedImage);
					}
				}
			}
		}
		
		resetDragState();
	}

	function updateDragImagePosition(x: number, y: number, element: HTMLElement) {
		element.style.left = `${x}px`;
		element.style.top = `${y}px`;
		element.style.transform = 'translate(-50%, -50%)';
	}

	function handleDrop(tierIndex: number | null) {
		if (!draggedImage || draggedSource === null || draggedIndex === null) return;
		// Remove from source
		if (draggedSource === 'uploaded') {
			uploadedImages = uploadedImages.filter((_, i) => i !== draggedIndex);
		} else if (draggedSource === 'tier' && draggedSourceTierIndex !== null) {
			tiers.update(t => {
				const newTiers = [...t];
				newTiers[draggedSourceTierIndex!].items = newTiers[draggedSourceTierIndex!].items.filter((_, i) => i !== draggedIndex);
				return newTiers;
			});
		} else if (draggedSource === 'category' && selectedCategory) {
			imageCategories[selectedCategory] = imageCategories[selectedCategory].filter((_, i) => i !== draggedIndex);
			
		}


		// Add to destination
		if (tierIndex !== null && tierIndex >= 0) {
			
			tiers.update(t => {
				const newTiers = [...t];
				if (newTiers[tierIndex]) {
					newTiers[tierIndex].items = [...newTiers[tierIndex].items, draggedImage as TierItem];
				}
				return newTiers;
			});
		} else {
			uploadedImages = [...uploadedImages, draggedImage as TierItem];
		}

		document.querySelectorAll('.tier-container').forEach(tier => {
						tier.classList.remove('border-2', 'border-dashed', 'border-white');
					});

		resetDragState();
	}

	function handleBinDrop() {
		if (draggedImage && draggedSource !== null && draggedIndex !== null) {
			if (draggedSource === 'uploaded') {
				uploadedImages = uploadedImages.filter((_, i) => i !== draggedIndex);
			} else if (draggedSource === 'tier' && draggedSourceTierIndex !== null) {
				tiers.update((t) => {
					const newTiers = [...t];
					newTiers[draggedSourceTierIndex as number].items = newTiers[draggedSourceTierIndex as number].items.filter((_, i) => i !== draggedIndex);
					return newTiers;
				});
			} else if (draggedSource === 'category' && selectedCategory) {
				imageCategories[selectedCategory] = imageCategories[selectedCategory].filter((_, i) => i !== draggedIndex);
			}
		}
		resetDragState();
	}

	function resetDragState() {
		isDragging.set(false);
		isOverBin.set(false);
		touchStartX = null;
		touchStartY = null;
		draggedImage = null;
		draggedSource = null;
		draggedIndex = null;
		draggedSourceTierIndex = null;

		document.querySelectorAll('.category-images').forEach(category => {
					category.classList.remove('bg-neutral-600');
				});
		
		// Reset any dragged image elements
		document.querySelectorAll('img[dragging="true"]').forEach(img => {
			const imageElement = img as HTMLElement;
			imageElement.removeAttribute('dragging');
			imageElement.classList.remove('fixed', 'w-24', 'h-24', 'pointer-events-none', 'z-50', 'cursor-grabbing');
			imageElement.style.left = '';
			imageElement.style.top = '';
			imageElement.style.transform = '';
		});
	}

	function handleDragOver(event: DragEvent) {
		event.preventDefault();
		if (event.dataTransfer) {
			event.dataTransfer.dropEffect = 'move';
		}

		// Auto-scroll functionality for mouse drag
		const scrollThreshold = 500; // pixels from top/bottom to trigger scroll
		const scrollSpeed = 10; // pixels per scroll
		const viewportHeight = window.innerHeight;
		const mouseY = event.clientY;

		// Scroll up if near top of viewport
		if (mouseY < scrollThreshold) {
			window.scrollBy(0, -scrollSpeed);
		}
		// Scroll down if near bottom of viewport
		else if (mouseY > viewportHeight - scrollThreshold) {
			window.scrollBy(0, scrollSpeed);
		}
	}

	function handleDragEnd() {
		isDragging.set(false);
	}

	function handleSubmit() {
		if ($tierListName.trim()) {
			showContent.set(true);
		}
	}

	function startEditing() {
		isEditing.set(true);
		showContent.set(false);
	}

	function selectCategory(category: string) {
		if (selectedCategory === category) {
			// If clicking the same category, close it and reset the button state
			selectedCategory = null;
			showCategoryImages = false;
		} else {
			// Otherwise, open the new category
			selectedCategory = category;
			showCategoryImages = true;
		}
	}

	function addCategoryImage(image: { src: string; name: string }) {
		uploadedImages = [...uploadedImages, image];
	}

	// In loadCategoryImages function
	async function loadCategoryImages(category: string) {
		// ...
		loadingCategoryId.set(category);
		// ...
		
		// At the end of the function
		loadingCategoryId.set(null);
	}
</script>

<div class="flex flex-col gap-5 p-6 sm:p-8 bg-neutral-900 min-h-screen font-['DM_Sans']">
	{#if !$showContent}
		<div class="flex flex-col items-center justify-center h-full my-auto">
			<div class="w-full max-w-md flex flex-col items-center justify-center">
				<img src="{logo}" alt="Creator Camp Logo" class=" mb-4">
				<h1 class="text-4xl font-bold text-center mb-12 text-white leading-tight">Tier List Maker</h1>
				<div class="flex gap-4">
					<input
						type="text"
						bind:value={$tierListName}
						placeholder="Tier list name"
						class="w-full p-3 text-xl bg-neutral-800 text-white rounded-lg border border-neutral-700 focus:border-blue-500 focus:outline-none"
					/>
					<button
						on:click={handleSubmit}
						class="px-6 py-3 text-white rounded-lg bg-[#28B9EB] hover:bg-[#219EC9] transition-colors"
					>
						OK
					</button>
				</div>
			</div>
		</div>
	{:else}
		<div class="mb-8 flex items-center gap-3">
			<h1 class="text-2xl font-bold text-white">{$tierListName}</h1>
			<button
				on:click={startEditing}
				class="p-2 text-gray-400 hover:text-white transition-colors"
				aria-label="Edit tier list name"
			>
				<svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
					<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15.232 5.232l3.536 3.536m-2.036-5.036a2.5 2.5 0 113.536 3.536L6.5 21.036H3v-3.572L16.732 3.732z" />
				</svg>
			</button>
		</div>
		{#each $tiers as tier, index}
			<div 
				class="flex items-center tier-container rounded-lg overflow-hidden relative" 
				data-index={index}
				role="listitem"
				on:drop={(e) => {
					e.preventDefault();
					e.stopPropagation();
					handleDrop(index);
				}} 
				on:dragover={handleDragOver}
				on:dragenter={(e) => {
					e.preventDefault();
					e.stopPropagation();
					document.querySelectorAll('.tier-container').forEach(tier => {
						tier.classList.remove('bg-neutral-800', 'h-24', 'border-2', 'border-dashed', 'border-white');
					});
					
					e.currentTarget.classList.add( 'bg-neutral-800', 'h-24', 'border-2', 'border-dashed', 'border-white');

				}}
				on:dragleave={(e) => {
					e.preventDefault();
					e.stopPropagation();

					
				}}
			>
				<div class="flex gap-2.5 flex-grow tier">
					<div class="flex gap-2.5 flex-grow flex-row tier bg-neutral-800" data-index={index} >
						<div class="w-12 h-full min-h-12 flex items-center justify-center text-2xl font-bold text-black mr-2.5" style="background-color: {tier.color}">
							{tier.name}
						</div>
						<div 
							class="flex-grow min-h-12 flex items-center gap-2.5 p-2"
							role="list"
							on:drop={(e) => {
								e.preventDefault();
								e.stopPropagation();
								handleDrop(index);
							}}
							on:dragover={(e) => {
								e.preventDefault();
								e.stopPropagation();
								e.dataTransfer!.dropEffect = 'move';
							}}
						>
							{#each tier.items as item, itemIndex}
								<div class="relative z-100">
									<img 
										src={item.src} 
										alt={item.name}
										class="w-24 h-24 object-contain cursor-pointer" 
										draggable="true" 
										on:dragstart={(event) => handleDragStart(event, item, 'tier', itemIndex, index)} 
										on:dragend={handleDragEnd} 
										on:touchstart={(event) => handleTouchStart(event, item, 'tier', itemIndex, index)} 
										on:touchmove={handleTouchMove} 
										on:touchend={handleTouchEnd} 
									/>
								</div>
							{/each}
						</div>
					</div>
				</div>
			</div>
		{/each}
		<div class="text-white">
			<div class="mb-6 category-images rounded-lg overflow-hidden">
				<div class="flex flex-wrap gap-2 mb-4" 
				>
					{#if $loadingCategories}
						<!-- Skeleton loaders for categories -->
						{#each Array(10) as _}
							<div class="h-10 w-24 rounded-lg skeleton"></div>
						{/each}
					{:else}
						{#each Object.entries(imageCategories) as [category]}
							<button
								on:click={() => selectCategory(category)}
								class="px-4 py-2 text-white rounded-lg transition-colors {selectedCategory === category ? 'bg-[#28B9EB] hover:bg-[#219EC9]' : 'bg-neutral-800 hover:bg-neutral-700'}"
							>
								{category}
							</button>
						{/each}
					{/if}
				</div>

				{#if selectedCategory}
					<div class="flex flex-wrap gap-2 category-images-container"
					role="list"
				on:dragover={(e) => {
					
					e.currentTarget.classList.add( 'bg-neutral-800', 'border-2', 'border-dashed', 'border-white');

				}}

				on:dragleave={(e) => {
					e.currentTarget.classList.remove( 'bg-neutral-800', 'border-2', 'border-dashed', 'border-white');

				}}
					>
						{#if imageCategories[selectedCategory] && imageCategories[selectedCategory].length > 0}
							{#each imageCategories[selectedCategory] as image, index}
								<div class="relative aspect-square w-24 h-24 bg-neutral-800 rounded-lg"
								>
									<img
										src={image.src}
										alt={image.name}
										loading="lazy"
										class="w-full h-full object-contain rounded-lg cursor-pointer"
										draggable="true"
										on:dragstart={(event) => handleDragStart(event, image, 'category', index)}
										on:dragend={handleDragEnd}
										on:touchstart={(event) => handleTouchStart(event, image, 'category', index)}
										on:touchmove={handleTouchMove}
										on:touchend={handleTouchEnd}
									/>
								</div>
							{/each}
						{:else if $loadingCategoryId === selectedCategory}
							<!-- Skeleton loaders with shimmer effect -->
							{#each Array(12) as _}
								<div class="w-24 h-24 rounded-lg skeleton"></div>
							{/each}
						{:else}
							<div class="w-full text-center py-6 text-neutral-400">
								No images found in this category
							</div>
						{/if}
					</div>
				{/if}
			</div>
			{#if uploadedImages.length > 0}
				<div class="mb-6">
					<div class="flex flex-wrap gap-2">
						{#each uploadedImages as image, index}
							<div class="relative aspect-square w-24 h-24">
								<img
									src={image.src}
									alt={image.name}
									class="w-full h-full object-contain rounded-lg cursor-pointer bg-white"
									draggable="true"
									on:dragstart={(event) => handleDragStart(event, image, 'uploaded', index)}
									on:dragend={handleDragEnd}
									on:touchstart={(event) => handleTouchStart(event, image, 'uploaded', index)}
									on:touchmove={handleTouchMove}
									on:touchend={handleTouchEnd}
								/>
							</div>
						{/each}
					</div>
				</div>
			{/if}
			<div class="flex gap-4 mb-4">
				<label class="inline-flex items-center px-4 py-2 bg-[#28B9EB] hover:bg-[#219EC9] text-white rounded-full cursor-pointer transition-colors duration-200">
					<svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 mr-2" fill="none" viewBox="0 0 24 24" stroke="currentColor">
						<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-8l-4-4m0 0L8 8m4-4v12" />
					</svg>
					Choose Files
					<input type="file" accept="image/*" multiple on:change={handleFileUpload} class="hidden" />
				</label>
			</div>
			
		</div>
		{#if $isDragging}
		<div 
			class="fixed right-0 top-0 h-48 w-24 border-l-2 rounded-bl-lg border-dashed text-center transition-colors duration-200 z-100 bg-red-100 {$isOverBin ? 'border-red-500 bg-red-300 text-red-500' : 'border-gray-300 text-red-900'} delete-bin" 
			role="button"
			tabindex="0"
			on:drop={handleBinDrop} 
			on:dragover={handleDragOver}
		>
			<p class="mt-4">Drag here to delete</p>
		</div>
	{/if}
	{/if}
</div>

<style>
	@keyframes shimmer {
		0% {
			background-position: -200% 0;
		}
		100% {
			background-position: 200% 0;
		}
	}
	
	.skeleton {
		background: linear-gradient(90deg, #2a2a2a 25%, #3a3a3a 50%, #2a2a2a 75%);
		background-size: 200% 100%;
		animation: shimmer 1.5s infinite;
	}
</style>