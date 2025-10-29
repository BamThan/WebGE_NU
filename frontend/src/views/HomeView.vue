<script setup>
import { RouterLink } from 'vue-router';

import Layout from '@/layout/Layout.vue';

</script>

<template>
  <Layout>
    <div class="text-center mt-30 md:m-30 ">
      <p class="text-xl md:text-3xl">แบบทดสอบนี้จะช่วยแนะนำว่าวิชาไหนเหมาะกับคุณ<br></br>เพียงตอบไม่กี่คำถามง่ายๆ</p>
      <RouterLink :to="'/matchcase'" class="btn bg-[#2E8B57] hover:bg-[#3CB371] text-white mt-4 md:mt-10 md:w-40 h-15 md:h-15 text-lg md:text-xl">เริ่มต้นใช้งาน</RouterLink>
    </div>
  </Layout>
</template>