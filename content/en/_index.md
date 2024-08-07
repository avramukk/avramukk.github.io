---
title: "Головна сторінка"
description: "Це моя головна сторінка з текстом"
---

<div class="content-container">
  <div class="description-container">
    My name is <a target="_blank" rel="noopener noreferrer" href="https://avramukk.com">Mykola Avramuk</a> <br>
    I`m Engineer from Ukraine<br>
    My area of interest: <br>
    • Video Streaming <br>
    • Quality Assurance<br>
    • Software Development<br>
    • DevOps and Cloud<br>
  </div>

  <div class="image-container">
    <img src="avatar.png" alt="My Image" class="center-image">
  </div>
</div>

{{< icon vendor="simple" name="github" link="https://github.com/avramukk" >}}

{{< icon vendor="simple" name="linkedin" link="https://www.linkedin.com/in/avramukk" >}}

<style>
.content-container {
  display: flex;
  justify-content: center;
  align-items: center;
}
.image-container {
  max-width: 500px;
}
.description-container {
  text-align: left;
  margin-right: 20px;
}
</style>
