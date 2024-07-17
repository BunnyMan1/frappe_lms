<template>
	<div v-if="courses.data">
	  <header class="sticky top-0 z-10 border-b bg-white px-3 py-2.5 sm:px-5">
		<!-- For desktop and laptop views -->
		<div class="hidden md:flex md:flex-row md:items-center md:justify-between">
        <div class="flex items-center space-x-4">
          <Breadcrumbs class="h-7" :items="[{ label: __('All Courses'), route: { name: 'Courses' } }]" />
        </div>
        <div class="flex space-x-2">
          <FormControl type="text" placeholder="Search Course" v-model="searchQuery" @input="courses.reload()">
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
			<FormControl type="text" placeholder="Search Course" v-model="searchQuery" @input="courses.reload()">
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
			<Autocomplete
			  :options="courseCategories"
			  v-model="selectedCategory"
			  placeholder="Course Category"
			  class="w-full"
			/>
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
				<Badge theme="gray">
				  {{ tab.count }}
				</Badge>
			  </button>
			</div>
		  </template>
		  <template #default="{ tab }">
			<div v-if="tab.courses && tab.courses.value.length"
			  class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-5 my-5 mx-5">
			  <router-link v-for="course in tab.courses.value" :to="course.membership && course.current_lesson
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
					}
				">
				<CourseCard :course="course" />
			  </router-link>
			</div>
			<div v-else class="grid flex-1 place-items-center text-xl font-medium text-gray-500">
			  <div class="flex flex-col items-center justify-center mt-4">
				<div>
				  {{ __('No {0} courses found').format(tab.label.toLowerCase()) }}
				</div>
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
	createResource,
	Autocomplete,
	createListResource
  } from 'frappe-ui'
  import CourseCard from '@/components/CourseCard.vue'
  import { Plus, Search } from 'lucide-vue-next'
  import { ref, computed, inject, watch, watchEffect } from 'vue'
  import { updateDocumentTitle } from '@/utils'
  
  const user = inject('$user')
  const searchQuery = ref('')
  const selectedCategory = ref('all')
  const courseCategories = ref([])

  const courses = createResource({
	url: 'lms.lms.utils.get_courses',
	cache: ['courses', user.data?.email],
	auto: true,
  })
  
  watch(selectedCategory, () => {
	courses.reload();
  });
  
  const tabIndex = ref(0)
  let tabs
  
  const makeTabs = computed(() => {
	tabs = []
	addToTabs('Live')
	addToTabs('New')
	addToTabs('Upcoming')
  
	if (user.data) {
	  addToTabs('Enrolled')
  
	  if (
		user.data.is_moderator ||
		user.data.is_instructor ||
		courses.data?.created?.length
	  ) {
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
	let filteredCourses = courses.data[type];

	if (selectedCategory.value && selectedCategory.value !== 'all') {
		filteredCourses = filteredCourses.filter(course => course.category === selectedCategory.value);
	}

	if (searchQuery.value) {
		let query = searchQuery.value.toLowerCase();
		filteredCourses = filteredCourses.filter(
			(course) =>
				course.title.toLowerCase().includes(query) ||
				course.short_introduction.toLowerCase().includes(query) ||
				course.tags.filter((tag) => tag.toLowerCase().includes(query)).length
		);
	}

	return filteredCourses;
  }
  
  const pageMeta = computed(() => {
	return {
	  title: 'Courses',
	  description: 'All Courses divided by categories',
	}
  })
  
  updateDocumentTitle(pageMeta)
  </script>
