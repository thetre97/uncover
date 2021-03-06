<template>
  <section class="section">
    <div class="columns is-centered">
      <div class="column is-one-third">
        <h1 class="title is-size-4">
          Create Image post
        </h1>
        <hr>
        <form
          @submit.prevent="insertImage"
          class="form">
          <div class="box">
            <b-field label="Title">
              <b-input
                v-model="image.title"
                placeholder="Beautiful Landscape"
                required />
            </b-field>
            <b-field label="Description">
              <b-input
                v-model="image.description"
                maxlength="200"
                type="textarea"
                placeholder="Lorem ipsum dolor amet lomo coloring book poke, raclette chartreuse freegan mustache dreamcatcher"
                required />
            </b-field>
          </div>
          <div class="box">
            <b-field v-if="!src">
              <b-upload
                v-if="!uploading"
                @input="setImage"
                drag-drop>
                <section class="section">
                  <div class="content has-text-centered">
                    <p>
                      <b-icon
                        icon="upload"
                        size="is-large" />
                    </p>
                    <p>Drop your file here or click to upload</p>
                  </div>
                </section>
              </b-upload>
              <b-progress
                v-if="uploading"
                :value="uploading" />
            </b-field>
            <figure
              v-if="src"
              class="image">
              <img
                :src="src"
                :alt="image.altText">
            </figure>
            <br>
            <b-field>
              <b-input
                v-model="image.altText"
                placeholder="Brief Image Description"
                required />
            </b-field>
          </div>
          <div class="field is-grouped is-grouped-right">
            <div class="control">
              <b-button
                type="is-primary"
                native-type="submit">
                Save
              </b-button>
            </div>
          </div>
        </form>
      </div>
    </div>
  </section>
</template>

<script>
import axios from 'axios'
// Queries
import INSERT_IMAGE_MUTATION from '@/graphql/Images/InsertImage.gql'
import ALL_IMAGES_QUERY from '@/graphql/Images/AllImages.gql'
export default {
  middleware: 'isAuth',
  data: () => ({
    src: '',
    uploading: 0,
    image: {
      title: '',
      description: '',
      altText: ''
    }
  }),
  methods: {
    async setImage (file) {
      const CLOUDINARY_NAME = process.env.CLOUDINARY_NAME
      const CLOUDINARY_PRESET = process.env.CLOUDINARY_PRESET
      const uploadUrl = `https://api.cloudinary.com/v1_1/${CLOUDINARY_NAME}/upload`
      try {
        const payload = new FormData()
        payload.append('upload_preset', CLOUDINARY_PRESET)
        payload.append('tags', 'browser_upload')
        payload.append('file', file)

        const { data } = await axios.post(uploadUrl, payload, {
          onUploadProgress: ({ loaded, total }) => {
            const progress = parseInt((loaded / total) * 100)
            this.uploading = progress
          }
        })
        this.src = data.secure_url
      } catch (error) {
        this.$buefy.toast.open({
          message: error.message,
          type: 'is-danger'
        })
      }
    },
    async insertImage () {
      const image = {
        user_id: this.$auth.user.id,
        url: this.src,
        ...this.image
      }
      try {
        if (!image.url) throw new Error('No Image')
        await this.$apollo.mutate({
          mutation: INSERT_IMAGE_MUTATION,
          variables: { image },
          update: (store, { data: { insert_images: { returning: [image] } } }) => {
            const data = store.readQuery({ query: ALL_IMAGES_QUERY, variables: { limit: 10 } })
            if (!data) return
            data.images.nodes.unshift(image)
            store.writeQuery({ query: ALL_IMAGES_QUERY, variables: { limit: 10 }, data })
          }
        })
        await this.$buefy.toast.open('Uploaded Image')
        this.$router.push('/')
      } catch (error) {
        this.$buefy.toast.open({
          message: error.message,
          type: 'is-danger'
        })
      }
    }
  }
}
</script>
