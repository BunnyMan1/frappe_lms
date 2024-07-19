<template>
	<div v-if="courses.data">
	  <header class="sticky top-0 z-10 border-b bg-white px-3 py-2.5 sm:px-5">
		<!-- For desktop and laptop views -->
		<div class="hidden md:flex md:flex-row md:items-center md:justify-between">
		  <div class="flex items-center space-x-4">
			<Breadcrumbs class="h-7" :items="[{ label: __('All Courses'), route: { name: 'Courses' } }]" />
			<Autocomplete :options="courseCategories" v-model="selectedCategory"
			  placeholder="Select Course Category" class="w-full" />
		  </div>
		  <div class="flex space-x-2">
			<FormControl type="text" placeholder="Search Course" v-model="searchQuery"
			  @input="courses.reload()">
			  <template #prefix>
				<Search class="w-4 stroke-1.5 text-gray-600" name="search" />
			  </template>
			</FormControl>
			<router-link :to="{
			  name: 'CreateCourse',
			  params: {
				courseName: 'new',
			  },
			}">
			  <Button v-if="user.data?.is_moderator" variant="solid">
				<template #prefix>
				  <Plus class="h-4 w-4" />
				</template>
				{{ __('New Course') }}
			  </Button>
			</router-link>
		  </div>
		</div>
  
		<!-- For mobile views -->
		<div class="flex flex-col space-y-2 md:hidden">
		  <div class="flex items-center space-x-4">
			<Breadcrumbs class="h-7" :items="[{ label: __('All Courses'), route: { name: 'Courses' } }]" />
		  </div>
		  <div class="flex space-x-2">
			<FormControl type="text" placeholder="Search Course" v-model="searchQuery"
			  @input="courses.reload()">
			  <template #prefix>
				<Search class="w-4 stroke-1.5 text-gray-600" name="search" />
			  </template>
			</FormControl>
			<router-link :to="{
			  name: 'CreateCourse',
			  params: {
				courseName: 'new',
			  },
			}">
			  <Button v-if="user.data?.is_moderator" variant="solid">
				<template #prefix>
				  <Plus class="h-4 w-4" />
				</template>
				{{ __('New Course') }}
			  </Button>
			</router-link>
		  </div>
		  <div>
			<Autocomplete :options="courseCategories" v-model="selectedCategory" placeholder="Course Category"
			  class="w-full" />
		  </div>
		</div>
	  </header>
	  <div>
		<Tabs v-model="tabIndex" tablistClass="overflow-x-visible flex-wrap !gap-3 md:flex-nowrap" :tabs="makeTabs">
		  <template #tab="{ tab, selected }">
			<div>
			  <button
				class="group -mb-px flex items-center gap-2 overflow-hidden border-b border-transparent py-2.5 text-base text-gray-600 duration-300 ease-in-out hover:border-gray-400 hover:text-gray-900"
				:class="{ 'text-gray-900': selected }">
				<component v-if="tab.icon" :is="tab.icon" class="h-5" />
				{{ __(tab.label) }}
				<Badge theme="gray">{{ tab.count }}</Badge>
			  </button>
			</div>
		  </template>
		  <template #default="{ tab }">
			<div v-if="tab.courses && tab.courses.value.length"
			  class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-5 my-5 mx-5">
			  <router-link v-for="course in tab.courses.value" :key="course.name" :to="course.membership && course.current_lesson
				? {
				  name: 'Lesson',
				  params: {
					courseName: course.name,
					chapterNumber: course.current_lesson.split('-')[0],
					lessonNumber: course.current_lesson.split('-')[1],
				  },
				}
				: course.membership
				  ? {
					name: 'Lesson',
					params: {
					  courseName: course.name,
					  chapterNumber: 1,
					  lessonNumber: 1,
					},
				  }
				  : {
					name: 'CourseDetail',
					params: { courseName: course.name },
					query: {
					  courseParams: JSON.stringify(courseParams.value),
					},
				  }">
				<CourseCard :course="course" />
			  </router-link>
			</div>
			<div v-else class="grid flex-1 place-items-center text-xl font-medium text-gray-500">
			  <div class="flex flex-col items-center justify-center mt-4">
				<div>{{ __('No {0} courses found').format(tab.label.toLowerCase()) }}</div>
			  </div>
			</div>
		  </template>
		</Tabs>
	  </div>
	</div>
  </template>
  
  <script setup>
  import {
	Breadcrumbs,
	Tabs,
	Badge,
	Button,
	FormControl,
	Autocomplete,
	createResource,
	createListResource,
  } from 'frappe-ui'
  import CourseCard from '@/components/CourseCard.vue'
  import { Plus, Search } from 'lucide-vue-next'
  import { ref, computed, inject, watchEffect, watch } from 'vue'
  import { updateDocumentTitle } from '@/utils'
  
  // Inject user data
  const user = inject('$user')
  
  // State variables
  const searchQuery = ref('')
  const selectedCategory = ref(null)
  
  // Define a reactive parameter object
  const courseParams = ref({
	category: null
  })
  
  // Resource to fetch courses
  const courses = createResource({
	url: 'lms.lms.utils.get_courses',
	params: courseParams.value,
	auto: true,
  })
  
  // Resource to fetch course categories
  const courseCategoriesResource = createListResource({
	doctype: 'LMS Course Category',
	fields: ['lms_category_name as label', 'lms_category_name as value'],
	orderBy: 'creation desc',
	start: 0,
	pageLength: 100, // Assuming you don't have more than 100 categories
	auto: true,
	transform: (data) => {
	  const transformedData = data.map(category => ({
		label: category.label,
		value: category.value
	  }));
	  return transformedData;
	}
  })
  
  const courseCategories = ref([])
  
  // Watch for changes in the fetched data and update courseCategories
  watchEffect(() => {
	if (courseCategoriesResource.data && courseCategoriesResource.data.length > 0) {
	  courseCategories.value = courseCategoriesResource.data;
	}
  })
  
  // Handle category selection change
  const filterCoursesByCategory = () => {
	// Update the category parameter and reload courses
	courseParams.value.category = selectedCategory.value?.value
	courses.reload()
  }
  
  // Watch for category changes and reload courses
  watch(selectedCategory, filterCoursesByCategory)
  
  // State for tabs
  const tabIndex = ref(0)
  let tabs
  
  const makeTabs = computed(() => {
	tabs = []
	addToTabs('All')
	addToTabs('Live')
	addToTabs('New')
	addToTabs('Upcoming')
  
	if (user.data) {
	  addToTabs('Enrolled')
  
	  if (user.data.is_moderator || user.data.is_instructor || courses.data?.created?.length) {
		addToTabs('Created')
	  }
  
	  if (user.data.is_moderator) {
		addToTabs('Under Review')
	  }
	}
	return tabs
  })
  
  const addToTabs = (label) => {
	let courses = getCourses(label.toLowerCase().split(' ').join('_'))
	tabs.push({
	  label,
	  courses: computed(() => courses),
	  count: computed(() => courses.length),
	})
  }
  
  const getCourses = (type) => {
	if (searchQuery.value) {
	  let query = searchQuery.value.toLowerCase()
	  return courses.data[type].filter(
		(course) =>
		  course.title.toLowerCase().includes(query) ||
		  course.short_introduction.toLowerCase().includes(query) ||
		  course.tags.filter((tag) => tag.toLowerCase().includes(query)).length
	  )
	}
	return courses.data[type]
  }
  
  const pageMeta = computed(() => {
	return {
	  title: 'Courses',
	  description: 'All Courses divided by categories',
	}
  })
  
  updateDocumentTitle(pageMeta)
  </script>
  