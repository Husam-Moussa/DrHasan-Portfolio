<script lang="ts">

	import { getGPUTier } from 'detect-gpu';
	import { onMount } from "svelte";
	import { fade } from "svelte/transition";
	import { ImageRenderer } from "$lib/effects/work-slider/renderer";
	import { letterSlideIn, letterSlideOut, maskSlideIn, maskSlideOut, workImageIntro, workListIntro } from "$lib/animations";
	import { loadPagePromise } from "$lib/store";
	import { dataState, scrollAnchorState, viewPortState, workScrollState } from "$lib/state.svelte";
	import { loadImage, onScrolledIntoView } from "$lib/utils";


	let workContainer: HTMLElement;
	let container: HTMLElement; 
	let listContainer: HTMLElement;
	let images: HTMLImageElement[] = [];
	let workItems: HTMLElement[] = [];

	let breakTitleWords: boolean = false;
	let currentActive: number = -1;
	let showcaseIndex: number = 0;

	let inViewResolve: (_: boolean) => void;
	const inViewPromise: Promise<boolean> = new Promise((resolve) => {
		inViewResolve = resolve;
	});


	class WorkSlider {

		currentMouseX = 0; 
		initialMouseX = 0;
		currentPosition = 0; 
		targetPosition = 0; 
		initialPosition = 0;
		offsetSpeed = 5000; 
		lerpSpeed = 0.1;

		onHold = (e: MouseEvent) => {
			e.preventDefault();
			if (currentActive >= 0 || workScrollState.active || (e.target! as HTMLElement).classList.contains("button")) return;

			this.initialMouseX = e.clientX;
			this.currentMouseX = e.clientX;
			workScrollState.active = true;

			if (workScrollState.active) {
				let style = window.getComputedStyle(listContainer);
				let matrix = new WebKitCSSMatrix(style.transform);

				this.initialPosition = matrix.m41;
			}
		}

		onRelease() {
			workScrollState.active = false;
		}
	
		onMouseMove = (e: MouseEvent) => {
			e.preventDefault();
			if (!workScrollState.active) return; 
			this.currentMouseX = e.clientX;

			let diff = (this.currentMouseX - this.initialMouseX) * -1;
			this.targetPosition = Math.round((this.initialPosition - (this.offsetSpeed * (diff / document.body.clientWidth))) * 100) / 100;
		}

		animate = () => {
			if (currentActive < 0) {
				let endPoint = listContainer.offsetWidth - document.body.clientWidth
				if (endPoint < 0) endPoint = listContainer.offsetWidth;

				if (this.targetPosition > 0) this.targetPosition = 0;
				if (this.targetPosition <= (endPoint * -1)) this.targetPosition = - endPoint;
			}

			this.currentPosition = this.lerp(this.currentPosition, this.targetPosition, this.lerpSpeed);
			
			workScrollState.speed = Math.round((this.currentPosition - this.targetPosition) * 100) / 100;
			listContainer.style.transform = `translate3d(${ Math.round(this.currentPosition * 100) / 100 }px, 0px, 0px)`;

			requestAnimationFrame(() => this.animate());
		}

		lerp(start: number, end: number, t: number) {
			return start * (1 - t) + end * t;
		}
	}

	
	const slider = new WorkSlider();

	onMount(async () => {

		onScrolledIntoView(workContainer, () => inViewResolve(true));

		const gpuTier = await getGPUTier();
		viewPortState.isMobile = gpuTier.isMobile!;

		await loadPagePromise;
		scrollAnchorState.work = workContainer;

		listContainer.style.transform = "translate3d(0px, 0px, 0px)";

		if (gpuTier.tier >= 2 && !gpuTier.isMobile && gpuTier.fps! >= 30) new ImageRenderer(container, images);
	});

	function toggleActiveItem(i: number) {
		currentActive = (currentActive == i) ? -1 : i;
		showcaseIndex = 0;
		if (currentActive >= 0) slider.targetPosition = -(workItems[i].offsetLeft - (window.innerWidth / 4) + window.innerWidth / 10);
	}

	function titleSlide(node: HTMLElement) {
		let title = letterSlideIn(node, { delay: 5, breakWord: false });
		title.anime({
			onComplete: () => breakTitleWords=true
		});
	}

	function getShowcaseImages(itemIndex: number) {
		// Return the cases array from your data structure
		return dataState.workData![itemIndex].cases || [];
	}

	function nextImage() {
		const maxImages = getShowcaseImages(currentActive).length;
		showcaseIndex = (showcaseIndex + 1) % maxImages;
	}

	function prevImage() {
		const maxImages = getShowcaseImages(currentActive).length;
		showcaseIndex = (showcaseIndex - 1 + maxImages) % maxImages;
	}

</script>




<div id="content-container" class="work-click-area" bind:this="{ workContainer }">

	<div class="content-wrapper" 
		role="listbox"
		tabindex="0"
		onmousedown={slider.onHold}
		onmouseup={slider.onRelease}
		onmouseleave={slider.onRelease}
		onmousemove={slider.onMouseMove}
		bind:this={container}
		class:disabled={currentActive >= 0}
		use:workListIntro={{ promise: inViewPromise, onComplete: async () => {
			const gpuTier = await getGPUTier();
			if (!gpuTier.isMobile) slider.animate();
		} }}
	>
		<div class:mobile={viewPortState.isMobile}>
			<ul class="work-list" 
				bind:this={listContainer} 
				class:hold={workScrollState.active}>

				{#each dataState.workData! as item, i}
					<li use:workImageIntro={{ promise: inViewPromise, delay: i*30 }}>
						<div class="list-item clickable passive" 
							class:ambient="{ currentActive !== i && currentActive >= 0 }" 
							class:active="{ currentActive === i }" 
							bind:this={ workItems[i] }>

							<div class="img-wrapper">
								{#await loadImage(`assets/imgs/work-back/${item.id}/cover.png`) then src}
									<img bind:this={images[i]} src="{src}" ondragstart={(e) => {e.preventDefault()}} draggable="false" alt="{item.title} Background">
								{/await}
							</div>
							{#await inViewPromise then _}
								<div class="text-top-wrapper" class:hidden={currentActive >= 0 || workScrollState.active}>
									<p 
										class="item-index"
										in:maskSlideIn={{
											delay: (i*30)+100,
											reverse: true
										}}>
										{(i.toString().length > 1) ? (i+1) : "0"+(i+1).toString()}
									</p>
								</div>
								
								<div class="text-wrapper" class:hidden={currentActive >= 0 || workScrollState.active}>
									<h1 
										class="item-title" 
										>
										<span in:maskSlideIn={{
											duration: 900, 
											delay: (i*30)+300,
											reverse: true 
										}}>
											{item.title}
										</span>
									</h1>

									<button 
										class="button item-link interactive"
										onclick={() => toggleActiveItem(i)}
										in:maskSlideIn={{
											duration: 900,
											delay: (i*30)+450,
											reverse: true
										}}>
										view
									</button>
								</div>
							{/await}
						</div>
					</li>
				{/each}
			</ul>
		</div>

		<!-- Active work item details with enhanced showcase -->
		{#if currentActive !== -1}
			<div class="details-container">
				
				<!-- Left Column: Text Content -->
				<div class="text-content" in:fade={{ duration: 500, delay: 200 }} out:fade={{ duration: 300 }}>
					<div class="top-section">
						<div class="meta-info">
							<div class="index-wrapper">
								<div in:maskSlideIn={{ reverse: true, delay: 100 }} out:maskSlideOut>
									<span class="index-number">
										{#if (currentActive < 9)}
											{"0"+(currentActive+1)}
										{:else}
											{currentActive+1}
										{/if}
									</span>
								</div>
							</div>
							<div class="divider" transition:fade={{ delay: 200 }}></div>
							<div class="summary-wrapper">
								<div in:maskSlideIn={{ reverse: true, delay: 150 }} out:maskSlideOut>
									<h6 class="summary">{dataState.workData![currentActive].details?.summary || dataState.workData![currentActive].title}</h6>
								</div>
							</div>
						</div>
					</div>
					
					<div class="mid-section">
						<h1 class="project-title" 
							use:titleSlide
							out:letterSlideOut
							class:breakTitleWords
							onintroend={() => setTimeout(() => breakTitleWords = true, 100)}
							onoutrostart={() => setTimeout(() => breakTitleWords = false, 100)}>
							{dataState.workData![currentActive].title}
						</h1>
					</div>
					
					<div class="bottom-section">
						<div class="description-wrapper">
							<div in:maskSlideIn={{ reverse: true, delay: 200 }} out:maskSlideOut>
								<p class="description">
									{dataState.workData![currentActive].details.description}
								</p>
							</div>
						</div>

						<div class="info-grid">
							{#if dataState.workData![currentActive].roles && dataState.workData![currentActive].roles.length > 0}
								<div class="roles-section">
									<div in:maskSlideIn={{reverse: true, delay: 250}} out:maskSlideOut>
										<p class="section-label">Roles</p>
									</div>
									<ul class="roles-list" in:maskSlideIn={{ reverse: true, delay: 300 }} out:maskSlideOut>
										{#each dataState.workData![currentActive].roles as role}
											<li>{role}</li>
										{/each}
									</ul>
								</div>
							{/if}

							{#if dataState.workData![currentActive].links && dataState.workData![currentActive].links.length > 0}
								<div class="links-section" in:maskSlideIn={{ reverse: true, delay: 350 }} out:maskSlideOut>
									{#each dataState.workData![currentActive].links as link}
										<a href={link.link} target="_blank" class="project-link">
											<span>{link.text}</span>
											<svg width="16" height="16" viewBox="0 0 16 16" fill="none">
												<path d="M3 13L13 3M13 3H5M13 3V11" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
											</svg>
										</a>
									{/each}
								</div>
							{/if}
						</div>
					</div>

					<button class="close-btn" onclick={() => toggleActiveItem(currentActive)}>
						<div in:maskSlideIn={{ reverse: true, delay: 100 }} out:maskSlideOut>
							<svg width="24" height="24" viewBox="0 0 24 24" fill="none">
								<path d="M18 6L6 18M6 6L18 18" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
							</svg>
						</div>
					</button>
				</div>

				<!-- Right Column: Image Showcase -->
				<div class="showcase-column" in:fade={{ duration: 500, delay: 300 }} out:fade={{ duration: 300 }}>
					<div class="showcase-container">
						<div class="main-image-wrapper">
							{#each getShowcaseImages(currentActive) as imgSrc, idx}
								{#if idx === showcaseIndex}
									<div class="main-image" in:fade={{ duration: 600 }} out:fade={{ duration: 400 }}>
										<img src={imgSrc} alt="Project showcase {idx + 1}" />
									</div>
								{/if}
							{/each}
						</div>

						<div class="showcase-controls">
							<button 
								class="control-btn prev-btn" 
								onclick={prevImage}
								in:maskSlideIn={{ delay: 500, reverse: true }}
								aria-label="Previous image">
								<svg width="20" height="20" viewBox="0 0 20 20" fill="none">
									<path d="M12 16L6 10L12 4" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
								</svg>
							</button>

							<div class="pagination" in:maskSlideIn={{ delay: 600, reverse: true }}>
								{#each getShowcaseImages(currentActive) as _, idx}
									<button 
										class="pagination-dot" 
										class:active={idx === showcaseIndex}
										onclick={() => showcaseIndex = idx}
										aria-label="Go to image {idx + 1}">
									</button>
								{/each}
							</div>

							<button 
								class="control-btn next-btn" 
								onclick={nextImage}
								in:maskSlideIn={{ delay: 500, reverse: true }}
								aria-label="Next image">
								<svg width="20" height="20" viewBox="0 0 20 20" fill="none">
									<path d="M8 4L14 10L8 16" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
								</svg>
							</button>
						</div>

						<div class="image-counter" in:maskSlideIn={{ delay: 700, reverse: true }}>
							<span class="current">{showcaseIndex + 1}</span>
							<span class="separator">/</span>
							<span class="total">{getShowcaseImages(currentActive).length}</span>
						</div>
					</div>
				</div>
			</div>
		{/if}
	</div>
</div>


<style lang="sass">

@use "../consts" as consts
@include consts.textStyles()

#content-container.work-click-area
	margin-top: 30vh

#content-container.work-click-area .content-wrapper
	display: flex
	flex-direction: column
	cursor: grab
	position: relative

	&.disabled
		cursor: default !important

		.mobile ul.work-list
			opacity: 0

	.mobile
		width: 100%
		height: 100%
		overflow-x: auto
	
	*
		-webkit-touch-callout: none
		-webkit-user-select: none
		-moz-user-select: none
		-ms-user-select: none
		user-select: none

	.details-container
		position: absolute
		left: 0
		top: 0
		height: 100%
		width: 100%
		display: grid
		grid-template-columns: 1fr 1fr
		gap: 6vw
		padding: 0 8vw
		box-sizing: border-box

		// Text Content Column
		.text-content
			display: flex
			flex-direction: column
			justify-content: space-between
			position: relative
			padding: 4vh 0

			.top-section
				.meta-info
					display: flex
					align-items: center
					gap: 2vw

					.index-number
						font-family: consts.$font
						font-size: 1.8vh
						letter-spacing: 0.3vh
						opacity: 0.7

					.divider
						width: 4vw
						height: 1px
						background: rgba(255, 255, 255, 0.3)

					.summary
						font-family: consts.$font
						font-size: 1.6vh
						text-transform: uppercase
						letter-spacing: 0.15vh
						font-weight: 400
						opacity: 0.85

			.mid-section
				margin: 6vh 0

				.project-title
					font-family: consts.$titleFont
					font-size: clamp(3rem, 6vw, 8rem)
					text-transform: lowercase
					font-weight: 400
					line-height: 0.95
					letter-spacing: -0.02em

					&.breakTitleWords
						display: inline-block
						max-width: min-content

			.bottom-section
				display: flex
				flex-direction: column
				gap: 4vh

				.description
					font-family: consts.$font
					font-size: 1.6vh
					line-height: 1.7
					opacity: 0.9
					max-width: 90%

				.info-grid
					display: grid
					grid-template-columns: 1fr 1fr
					gap: 4vh
					margin-top: 2vh

					.section-label
						font-family: consts.$font
						font-size: 1.3vh
						text-transform: uppercase
						letter-spacing: 0.3vh
						opacity: 0.6
						margin-bottom: 2vh

					.roles-list
						list-style: none
						display: flex
						flex-direction: column
						gap: 1.2vh

						li
							font-family: consts.$font
							font-size: 1.6vh
							opacity: 0.85
							text-transform: uppercase
							letter-spacing: 0.1vh
							position: relative
							padding-left: 1.5vh

							&:before
								content: "+"
								position: absolute
								left: 0
								opacity: 0.5

					.links-section
						display: flex
						flex-direction: column
						gap: 2vh
						align-items: flex-end

						.project-link
							display: inline-flex
							align-items: center
							gap: 1vh
							font-family: consts.$font
							font-size: 1.4vh
							text-transform: uppercase
							letter-spacing: 0.2vh
							text-decoration: none
							color: white
							padding: 1.5vh 2.5vh
							border: 1px solid rgba(255, 255, 255, 0.2)
							transition: all 0.3s cubic-bezier(0.25, 1, 0.5, 1)
							position: relative
							overflow: hidden

							&:before
								content: ""
								position: absolute
								top: 0
								left: -100%
								width: 100%
								height: 100%
								background: white
								transition: left 0.4s cubic-bezier(0.25, 1, 0.5, 1)
								z-index: -1

							&:hover
								color: black
								border-color: white

								&:before
									left: 0

								svg
									transform: translateX(3px) translateY(-3px)

							svg
								transition: transform 0.3s ease

			.close-btn
				position: absolute
				top: 2vh
				right: 0
				background: transparent
				border: none
				color: white
				cursor: pointer
				padding: 1.5vh
				opacity: 0.7
				transition: all 0.3s ease

				&:hover
					opacity: 1
					transform: rotate(90deg)

		// Showcase Column
		.showcase-column
			display: flex
			align-items: center
			justify-content: center
			padding: 4vh 0

			.showcase-container
				width: 100%
				height: 100%
				display: flex
				flex-direction: column
				justify-content: center
				position: relative

				.main-image-wrapper
					position: relative
					width: 100%
					height: 65vh
					display: flex
					align-items: center
					justify-content: center
					background: rgba(255, 255, 255, 0.02)
					border: 1px solid rgba(255, 255, 255, 0.08)
					backdrop-filter: blur(10px)
					border-radius: 4px
					overflow: hidden

					.main-image
						position: absolute
						width: 100%
						height: 100%
						display: flex
						align-items: center
						justify-content: center
						padding: 3vh

						img
							max-width: 100%
							max-height: 100%
							object-fit: contain
							filter: drop-shadow(0 10px 40px rgba(0, 0, 0, 0.4))

				.showcase-controls
					display: flex
					align-items: center
					justify-content: center
					gap: 3vw
					margin-top: 4vh

					.control-btn
						width: 3.5vw
						height: 3.5vw
						min-width: 40px
						min-height: 40px
						display: flex
						align-items: center
						justify-content: center
						background: transparent
						border: 1px solid rgba(255, 255, 255, 0.2)
						color: white
						cursor: pointer
						transition: all 0.3s cubic-bezier(0.25, 1, 0.5, 1)
						border-radius: 50%

						&:hover
							background: white
							color: black
							border-color: white
							transform: scale(1.1)

						&:active
							transform: scale(0.95)

					.pagination
						display: flex
						gap: 1vw
						align-items: center

						.pagination-dot
							width: 0.6vw
							height: 0.6vw
							min-width: 8px
							min-height: 8px
							border-radius: 50%
							background: transparent
							border: 1px solid rgba(255, 255, 255, 0.3)
							cursor: pointer
							transition: all 0.3s cubic-bezier(0.25, 1, 0.5, 1)
							padding: 0

							&.active
								background: white
								border-color: white

							&:hover:not(.active)
								border-color: white
								transform: scale(1.2)

				.image-counter
					position: absolute
					bottom: 2vh
					right: 2vh
					font-family: consts.$font
					font-size: 1.4vh
					letter-spacing: 0.2vh
					opacity: 0.5

					.separator
						margin: 0 0.5vh
						opacity: 0.5

		// Tablet Breakpoint
		@media only screen and (max-width: 1200px)
			grid-template-columns: 1fr
			gap: 6vh
			padding: 4vh 10vw

			.text-content
				.mid-section
					.project-title
						font-size: clamp(2.5rem, 10vw, 6rem)

				.bottom-section
					.info-grid
						grid-template-columns: 1fr

			.showcase-column
				.showcase-container
					.main-image-wrapper
						height: 50vh

					.showcase-controls
						.control-btn
							width: 50px
							height: 50px

						.pagination .pagination-dot
							width: 10px
							height: 10px

		// Mobile Breakpoint
		@media only screen and (max-width: 750px)
			grid-template-columns: 1fr
			padding: 3vh 8vw

			.text-content
				padding: 2vh 0

				.top-section
					.meta-info
						.index-number
							font-size: 1.4vh

						.summary
							font-size: 1.3vh

				.mid-section
					margin: 4vh 0

					.project-title
						font-size: clamp(2rem, 12vw, 4rem)

				.bottom-section
					gap: 3vh

					.description
						font-size: 1.5vh
						max-width: 100%

					.info-grid
						gap: 3vh

						.links-section
							align-items: flex-start

							.project-link
								font-size: 1.6vh
								padding: 2vh 3vh

				.close-btn
					top: 1vh
					padding: 2vh

			.showcase-column
				.showcase-container
					.main-image-wrapper
						height: 40vh

					.showcase-controls
						gap: 6vw
						margin-top: 3vh

						.control-btn
							width: 45px
							height: 45px

						.pagination .pagination-dot
							width: 12px
							height: 12px

	ul.work-list
		margin-top: auto
		margin: auto 0
		padding: 0 5vw
		list-style-type: none
		display: flex
		flex-direction: row
		align-items: center
		height: 75vh
		min-width: min-content
		opacity: 1
		transition: opacity 0.5s ease
		-webkit-transition: opacity 0.5s ease

		&.hold
			.list-item
				height: 45vh !important

		.list-item
			display: inline-flex
			justify-content: flex-end
			overflow: hidden
			height: 55vh
			width: 23vw
			box-sizing: border-box
			position: relative
			overflow: hidden
			z-index: 3
			margin-right: 6vw
			transition: width 0.7s cubic-bezier(0.25, 1, 0.5, 1), height 0.7s cubic-bezier(0.25, 1, 0.5, 1), margin 0.8s cubic-bezier(0.25, 1, 0.5, 1)

			*
				transition: opacity 0.3s ease
				-webkit-transition: opacity 0.3s ease

			&.active
				height: 60vh
				width: 50vw
				margin-right: 16vw
				margin-left: 10vw

				.img-wrapper
					width: 100%

			&.ambient
				height: 45vh

			.hidden
				opacity: 0

			.img-wrapper
				overflow: hidden
				height: 100%
				z-index: 1
				position: relative
				width: 85%
				margin-right: 15%
				box-shadow: 3px 9px 18px rgba(0, 0, 0, 0.2)
				
				img
					height: 110%
					width: 110%
					object-fit: cover
					position: absolute
					top: 50%
					left: 50%
					transform: translate(-50%, -50%)
					-webkit-transform: translate(-50%, -50%)
					opacity: 0.5

			.text-top-wrapper
				position: absolute
				top: 6vh
				right: 0
				z-index: 2
				word-wrap: break-word
				white-space: normal
				text-align: right

				.item-index
					font-size: 1vw
					letter-spacing: 0.1vw
					text-transform: uppercase
					font-family: consts.$font

			.text-wrapper
				box-sizing: border-box
				display: flex
				flex-direction: column
				justify-content: flex-end
				position: absolute
				bottom: 10vh
				right: 0
				text-align: right
				z-index: 2

				.button
					font-size: 1.3vw
					letter-spacing: 0.1vw
					margin-top: 2vh
					text-transform: uppercase

				.item-title
					font-family: consts.$font
					font-weight: normal
					font-size: 2.5vw
					z-index: 0
					opacity: 1
					letter-spacing: 0.1vw
					line-height: 110%
					word-spacing: 80vw
					text-transform: lowercase
					word-wrap: break-word
					white-space: normal


				.inline-wrapper
					width: 100%
					position: relative
					margin-top: 1vh

					*
						display: inline-block

		@media only screen and (max-width: 1450px)
			.list-item
				width: 25vw

		@media only screen and (max-width: 1110px)
			.list-item
				width: 40vw

				.text-top-wrapper
					.item-index
						font-size: 2vh

				.text-wrapper
					width: calc(55vw - 10vh)

					.item-title
						font-size: 5vw

					.item-link
						font-size: 2vh

		@media only screen and (max-width: 650px)
			.list-item
				width: 75vw

				.text-wrapper
					width: calc(70vw - 10vh)

					.item-title
						font-size: 4.5vh

</style>