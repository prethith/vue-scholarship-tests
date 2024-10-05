<script setup>
  import { ref, onMounted } from 'vue';
  import { supabase } from './lib/supabaseClient'

  const students = ref([]);
  const scholarships = ref([]);
  const assignedScholarships = ref([]);
  const acceptedScholarships = ref([]);
  // const choiceMade = ref(false);

  async function getStudents() {
    const { data, error } = await supabase.from('Students').select().order('student_id', { ascending: true});  
    students.value = data;
    // console.log(students.value);
  }

  async function getScholarships() {
    const { data, error2 } = await supabase.from('Scholarships').select();
    scholarships.value = data;
  }


  function isEligible(student, scholarship) {
  // Initialize a flag to true, assuming the student is eligible
    let eligible = true;
    let atLeastOneConditionChecked = false; // A flag to ensure at least one condition is checked

    // Check if the student's rank meets the requirement
    if (scholarship.min_rank !== null) {
      atLeastOneConditionChecked = true;
      if (student.rank > scholarship.min_rank) {
        eligible = false; // If student's rank is higher than allowed, they're ineligible
      }
    }

    // Check if the student's category matches the requirement
    if (scholarship.category_req !== null) {
      atLeastOneConditionChecked = true;
      if (student.category_ID !== scholarship.category_req) {
        eligible = false; // If student's category doesn't match, they're ineligible
      }
    }

    // Check if the student's income is within the allowed limit
    if (scholarship.max_income !== null) {
      atLeastOneConditionChecked = true;
      if (student.income > scholarship.max_income) {
        eligible = false; // If student's income exceeds the maximum, they're ineligible
      }
    }

    // Check if the student's marks meet the requirement
    if (scholarship.min_marks !== null) {
      atLeastOneConditionChecked = true;
      if (student.marks_12th < scholarship.min_marks) {
        eligible = false; // If student's marks are below the minimum, they're ineligible
      }
    }

    // If no conditions were applicable (all were null), return false
    if (!atLeastOneConditionChecked) {
      eligible = false;
    }

    return eligible;
  }


  function getEligibleScholarships(student) {
    const eligibleScholarships = scholarships.value.filter(scholarship => {
      const result = isEligible(student, scholarship);
      return result;
    });
    return eligibleScholarships;
  }

  async function acceptScholarship(student, scholarship) {
    if (!acceptedScholarships.value[student.student_id]) {
      // Initialize the array if it doesn't exist
      acceptedScholarships.value[student.student_id] = [];
    }

    // Check if the scholarship is already in the accepted array
    const alreadyAccepted = acceptedScholarships.value[student.student_id].some(
      (acceptedScholarship) => acceptedScholarship.scholarship_id === scholarship.scholarship_id
    );

    if (!alreadyAccepted) {
      // Add the scholarship to the student's accepted scholarships array
      acceptedScholarships.value[student.student_id].push(scholarship);
    }
    console.log(acceptedScholarships.value);
  }

  async function rejectScholarship(student, scholarship) {
    // if the given scholarship already exists in the accepted shcolarship list
    if(acceptedScholarships.value[student.student_id].find(item => item.scholarship_id === scholarship.scholarship_id)) {
      // remove that scholarship from the list
      const updatedScholarships = acceptedScholarships.value[student.student_id].filter(item => item.scholarship_id !== scholarship.scholarship_id);
      acceptedScholarships.value[student.student_id] = [...updatedScholarships];

      const globalScholarship = scholarships.value.find(
        sch => sch.scholarship_id === scholarship.scholarship_id
      );

      // if found in the list, increase its capacity
      if (globalScholarship) {
        // console.log(`Initial capacity of ${globalScholarship.scholarship_name} is ${globalScholarship.capacity}`)
        globalScholarship.capacity++;
      }

      // await assignScholarships(students, scholarships);
    }
  }
  // loop through the list of students

  async function assignScholarships(students, scholarships) {

    // Ensure data is fetched before processing
    await Promise.all([getStudents(), getScholarships()]); // Wait for both fetches

    // Check if students and scholarships have valid data
    if (!students.value || !Array.isArray(students.value) || students.value.length === 0) {
        console.error("No students data available");
        return;
    }
    
    if (!scholarships.value || !Array.isArray(scholarships.value) || scholarships.value.length === 0) {
        console.error("No scholarships data available");
        return;
    }

    scholarships.value.forEach(scholarship => {
      // this is to handle the case when no of eligible students > capacity of scholarship
      if(scholarship.sorting_criteria === 1) {
        // sort students by rank
        students.value.sort((a,b) => a.rank - b.rank );
      } else if (scholarship.sorting_criteria === 2) {
        // sort students by income
        students.value.sort((a,b) => a.income - b.income);
      }

      console.log(`Students sorted for ${scholarship.scholarship_name}:`, students.value); // Debug: Verify sorting

      // Loop through sorted students to assign the scholarship
      students.value.forEach(student => {
        if (isEligible(student, scholarship) && scholarship.capacity > 0) {
          // Assign scholarship
          console.log(`${student.name} is assigned to ${scholarship.scholarship_name}`);

          // If the student doesn't have any assigned scholarships, initialize it
          if (!assignedScholarships.value[student.student_id]) {
            assignedScholarships.value[student.student_id] = [];
          }

          // Check if the scholarship is already assigned to avoid duplicates
          const alreadyAssigned = assignedScholarships.value[student.student_id].some(
            assigned => assigned.scholarship_id === scholarship.scholarship_id
          );

          if (!alreadyAssigned) {
            // Assign the scholarship
            assignedScholarships.value[student.student_id].push({
              scholarship_id: scholarship.scholarship_id,
              scholarship_name: scholarship.scholarship_name,
              capacity: scholarship.capacity,
            });

            // Decrease the scholarship capacity
            scholarship.capacity--;

            // Break out of the student loop for this scholarship once assigned
            return true;
          }
        }
      });
    });

    console.log(assignedScholarships.value);
  }


  onMounted(() => {
    getStudents();
    getScholarships();  
    assignScholarships(students, scholarships);
  });

</script>

<template>
  <table class="studentsList">
    <thead>
      <tr>
        <th class="studentid">StudentID</th>
        <th class="name">Name</th>
        <th class="eligible">Eligible Scholarships</th>
        <th class="assignedHeading">Assigned Scholarships</th>
        <th class="accepted">Accepted Scholarships</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="student in students.slice().sort((a,b) => a.student_id - b.student_id)" :key="student.student_id">
        <td>{{ student.student_id }}</td>
        <td>{{ student.name }}</td>
        <td>
          <ul>
            <li v-for="scholarship in getEligibleScholarships(student)" :key="scholarship.scholarship_id">
              {{ scholarship.scholarship_name }}
            </li>
          </ul>
          <p v-if="getEligibleScholarships(student).length === 0">
            Ineligible
          </p>
        </td>
        <td>
          <ul>
            <li v-for="scholarship in assignedScholarships[student.student_id]" :key="scholarship.scholarship_id" class="assignedList">
              {{ scholarship.scholarship_name }}
              <button @click="acceptScholarship(student, scholarship)">✓</button>
              <button @click="rejectScholarship(student, scholarship)" class="wrong">⨯</button>
            </li>
          </ul>
          <p v-if="!assignedScholarships[student.student_id]">
            Not assigned
          </p>
        </td>

        <td>
          <ul v-if="acceptedScholarships[student.student_id] && acceptedScholarships[student.student_id].length > 0">
            <li v-for="scholarship in acceptedScholarships[student.student_id]" :key="scholarship.scholarship_id">
                {{ scholarship.scholarship_name }}
            </li>
          </ul>
          <span v-else>
            None accepted
          </span>
        </td>
      </tr>
    </tbody>
  </table>

  <ul>
    <li v-for="scholarship in scholarships" :key="scholarship.scholarship_id">
      {{ scholarship.scholarship_name }} {{ scholarship.capacity }}
    </li>
  </ul>
</template>


<style scoped>
  li {
    list-style: none;
  }

  table {
    width: 80%;
    border: 1px solid #bfbfbf;
    border-radius: 12px;
    padding: 20px;
    margin: 10px;
    table-layout: fixed;
  }

  th,td {
    padding: 0;
    text-align: center;
  }

  th.studentid {
    width: 5%;
  }

  th.name {
    width: 10%;
  }

  th.eligible {
    width: 30%;
  }

  th.assignedHeading {
    width: 20%;
  }
  th.accepted { 
    width: 20%;
  }
  
  button {
    padding: 5px;
    margin: 5px;
  }
  .wrong {
    padding: 3px 6px;
  }
  .assignedList {
    display: block;
  }
</style>