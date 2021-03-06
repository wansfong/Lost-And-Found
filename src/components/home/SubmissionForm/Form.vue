<template>
    <v-card>
        <v-card-text>
            <v-container grid-list-md>
                <v-form v-model="valid">
                  <v-layout wrap>
                    <v-flex xs12>
                        <v-text-field
                          class="text-field"
                          v-model="type"
                          label="Item *"
                          :hint="typeHint"
                          counter="10"
                          :maxlength="10"
                          required
                          :error-messages="typeErrors"
                          @input="$v.type.$touch()"
                          @blur="$v.type.$touch()"
                        ></v-text-field>
                    </v-flex>
                    <v-flex xs12>
                        <v-textarea
                          v-model="description"
                          :hint="descriptionHint"
                          label="Item Description *"
                          counter="150"
                          :maxlength="150"
                          required
                          :error-messages="descriptionErrors"
                          @input="$v.description.$touch()"
                          @blur="$v.description.$touch()"
                        ></v-textarea>
                    </v-flex>
                    <v-flex xs12>
                        <v-text-field
                          v-model="contactEmail"
                          label="Contact Information *"
                          :hint="contactHint"
                          required
                          :error-messages="contactEmailErrors"
                          @input="$v.contactEmail.$touch()"
                          @blur="$v.contactEmail.$touch()"
                        ></v-text-field>
                    </v-flex>
                    <date-picker></date-picker>
                    <time-picker></time-picker>
                    <v-flex xs12>
                        <v-tabs
                          centered
                          v-model="active"
                          color="cyan"
                          dark
                          icons-and-text
                        >
                          <v-tabs-slider color="yellow"></v-tabs-slider>
                          <v-tab href="#tab-1">
                            Image Upload
                            <v-icon>fas fa-laptop</v-icon>
                          </v-tab>

                          <v-tab href="#tab-2">
                            Image URL
                            <v-icon>fas fa-wifi</v-icon>
                          </v-tab>

                          <v-tab-item id="tab-1">
                            <v-card flat>
                              <v-card-text>
                                <input type="file" accept=".jpg, .png, .gif" @change="getPicInfo">
                              </v-card-text>
                            </v-card>
                          </v-tab-item>
                          <v-tab-item id="tab-2">
                            <v-card flat>
                              <v-card-text>
                                <v-text-field label="URL" v-model="imageURL"></v-text-field>
                              </v-card-text>
                            </v-card>
                          </v-tab-item>
                        </v-tabs>
                    </v-flex>
                </v-layout>
                </v-form>
            </v-container>
            <br />
            <small>* indicates required field</small>
        </v-card-text>
        <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn
              @click="uploadImageAndDoc"
              :disabled="!valid"
            >Submit</v-btn>
            <v-btn
              color="cyan"
              dark
              @click.native="toggleSubmission"
            >Close</v-btn>
        </v-card-actions>
    </v-card>
</template>

<script>
// import components
import DatePicker from './DatePicker'
import TimePicker from './TimePicker'

// import event bus for child to parent communication
import { EventBus } from '../../../main'

// import to use vuelidate
import { email, required } from 'vuelidate/lib/validators'
import formMixin from '../../../mixins/form'

export default {
  components: {
    'date-picker': DatePicker,
    'time-picker': TimePicker
  },
  mixins: [formMixin],
  // vuelidate package allows us to include validations to enforce correct input format
  // https://monterail.github.io/vuelidate/#sub-builtin-validators
  validations: {
    type: {
      required
    },
    contactEmail: {
      required,
      email
    },
    description: {
      required
    }
  },
  computed: {
    // with functions with "...Errors": if errors array includes an element,
    // cause the input to project the error
    typeErrors () {
      const errors = []
      if (!this.$v.type.$dirty) return errors
      !this.$v.type.required && errors.push('type is required.')
      return errors
    },
    descriptionErrors () {
      const errors = []
      if (!this.$v.description.$dirty) return errors
      !this.$v.description.required && errors.push('description is required.')
      return errors
    },
    contactEmailErrors () {
      const errors = []
      if (!this.$v.contactEmail.$dirty) return errors
      !this.$v.contactEmail.email && errors.push('Must be valid e-mail')
      !this.$v.contactEmail.required && errors.push('E-mail is required')
      return errors
    }
  },
  props: [
    'db',
    'firebase',
    'user',
    'lat',
    'lng',
    'activeParent',
    'typeHint',
    'descriptionHint',
    'contactHint',
    'collectionName'
  ],
  data () {
    return {
      type: null,
      description: null,
      contactEmail: null,
      date: null,
      time: null,
      imageFile: null,
      imageURL: null,
      active: null,
      valid: true
    }
  },
  methods: {
    uploadImageAndDoc () {
      this.$v.$touch()
      if (!this.valid) {
        console.log('test')
        console.log('this.valid :', this.valid)
      }
      this.imageFile && this.active === 'tab-1' ? this.uploadPic(this.collectionName) : this.addDoc(this.collectionName)
      this.toggleSubmission()
    },
    addDoc (collectionName) {
      if (this.type) {
        this.feedback = null
        this.db.collection(collectionName).add({
          type: this.type,
          description: this.description,
          contactEmail: this.contactEmail,
          location: new this.firebase.firestore.GeoPoint(this.lat, this.lng),
          timestamp: this.date + ' ' + this.time,
          picture: this.imageURL,
          userID: this.user.uid
        }).then((docRef) => {
          console.log('doc :', docRef)
          docRef.get().then((doc) => {
            this.$store.dispatch('updateUserCollection', collectionName)
            this.$store.dispatch('updateCollection', collectionName)
          })
        })
          .catch(function (error) {
            console.log(error)
          })
      } else {
        this.feedback = 'You must enter an item type'
      }
    },
    /* updates the picture info in data */
    getPicInfo (e) {
      this.imageFile = e.target.files[0]
    },
    /* upload picture to Storage and save the url to data.imageURL */
    /* NOTE!!! must be called before addDoc() if user is including a picture */
    uploadPic (collectionName) {
      var name = this.user.uid + '-' + (+new Date()) + '-' + this.type // give picture unique name based on userID, timestamp, and item type
      console.log('uploadPic is running')
      // var metadata = { contentType: this.imageFile.type }
      const STORAGE = this.firebase.storage().ref()
      var uploadTask = STORAGE.child(name).putString(this.imageFile, 'data_url')
      var self = this
      uploadTask.on('state_changed', function (snapshot) {
        // Get task progress, including the number of bytes uploaded and the total number of bytes to be uploaded
        var progress = (snapshot.bytesTransferred / snapshot.totalBytes) * 100
        console.log('Upload is ' + progress + '% done')
      }, function (error) {
        // Handle unsuccessful uploads
        console.log("Error: couldn't upload picture,", error)
      }, function () {
        // Handle successful uploads on complete
        uploadTask.snapshot.ref.getDownloadURL().then(function (downloadURL) {
          self.imageURL = downloadURL
          self.addDoc(collectionName)
        })
      })
    },
    // close form
    toggleSubmission () {
      this.$v.$reset()
      EventBus.$emit('toggleSubmission')
    }
  },
  watch: {
    // reset every input if toggle between lost form and found form
    activeParent () {
      this.$v.$reset()
      this.type = null
      this.description = null
      this.contactEmail = this.user.email
      this.imageFile = null
      this.imageURL = null
    }
  },
  created () {
    // if date is changed from the child component, update the value here
    EventBus.$on('setDate', (date) => {
      this.date = date
    }
    )
    // if time is changed from the child component, update the value here
    EventBus.$on('setTime', (time) => {
      this.time = time
    }
    )
  }
}
</script>

<style>
</style>
