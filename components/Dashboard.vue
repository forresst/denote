<template>
  <div class="container">
    <div class="main" v-if="sortedNotes">
      <div class="columns">
        <div class="column is-4">
          <div class="sticky">
            <div class="add-note" v-if="addNote">
              <ComposeBox :handleSave="handleSaveNote" :handleCancel="handleCancelSave" />
            </div>
            <div class="add-note-trigger" v-else @click="addNote = true">
              <Edit3Icon />{{ $inter.formatMessage({ path: 'editor.addNote' }) }}
            </div>
            <div class="links">
              <span>
                <a
                  href="https://github.com/egoist/denote/issues/new"
                  target="_blank"
                >{{ $inter.formatMessage({ path: 'app.reportBugs' }) }}</a>
              </span>
              <span>
                <a
                  href="https://patreon.com/egoist"
                  target="_blank"
                >{{ $inter.formatMessage({ path: 'app.donate' }) }}</a>
              </span>
            </div>
          </div>
        </div>
        <div class="column is-8">
          <div
            class="card is-link"
            v-if="oldNotes && oldNotes.length > 0"
            @click="importOldNotes(oldNotes)"
          >{{ importingOldNotes ? 'Importing' : 'Import' }} {{ oldNotes.length }} old notes.</div>
          <div class="notes">
            <template v-for="note in sortedNotes">
              <div class="card note-editor" :key="note.id" v-if="editingIds.indexOf(note.id) > -1">
                <ComposeBox
                  :note="note"
                  :handleSave="handleEditNote"
                  :handleCancel="handleCancelEditNote"
                />
              </div>
              <div class="note" :key="note.id" v-else>
                <div class="note-title" v-if="note.title">{{ note.title }}</div>
                <div class="note-content" v-html="toHTML(note.content)"></div>
                <div class="note-meta">
                  <span class="note-date">{{ formatDate(new Date(note.createdAt)) }}</span>
                  <span class="note-action" @click="editingIds.push(note.id)">
                    {{ $inter.formatMessage({ path: 'note.edit' }) }}
                  </span>
                  <span
                    class="note-action"
                    @click="deleteNote(note.id)"
                  >{{ $inter.formatMessage({path: deletingId === note.id ? 'note.deleting' : 'note.delete' }) }}</span>
                </div>
              </div>
            </template>
          </div>
        </div>
      </div>
    </div>
    <div class="main" v-else>
      <div class="loading">
        <div></div>
      </div>
    </div>
  </div>
</template>

<script>
import Vue from 'vue'
import marked from 'marked'
import { userSession } from '../utils/userSession'
import { uid } from '../utils/uid'
import tinydate from 'tinydate'
import { Edit3Icon, XSquareIcon } from 'vue-feather-icons'
import ComposeBox from './ComposeBox.vue'
import { NOTES_FILE } from '../utils/constants'

const formatDate = tinydate('{YYYY}-{MM}-{DD} {HH}:{mm}:{ss}')

async function updateNotesFile(notes, data) {
  await userSession.putFile(
    NOTES_FILE,
    JSON.stringify({
      notes,
      updatedAt: new Date(),
      ...data
    })
  )
}

async function fetchOldNotes() {
  const file = await userSession.getFile('notes.json')
  return JSON.parse(file || '[]')
}

export default Vue.extend({
  components: {
    Edit3Icon,
    XSquareIcon,
    ComposeBox
  },

  data() {
    return {
      addNote: false,
      notes: null,
      title: '',
      content: '',
      saving: false,
      deletingId: null,
      editingIds: [],
      oldNotes: null,
      importingOldNotes: false
    }
  },

  async mounted() {
    const file = await userSession.getFile(NOTES_FILE)
    const { notes, disableOldNotes } = file ? JSON.parse(file) : { notes: [] }
    this.notes = notes

    if (!disableOldNotes) {
      fetchOldNotes().then(notes => {
        this.oldNotes = notes
      })
    }
  },

  methods: {
    formatDate,

    signout() {
      userSession.signUserOut('/')
    },

    async handleSaveNote(note, existing) {
      let newNotes

      // Update notes list
      if (existing) {
        newNotes = this.notes.map(_note => {
          if (_note.id === note.id) {
            return note
          }
          return _note
        })
      } else {
        newNotes = this.notes.concat(note)
      }

      await updateNotesFile(newNotes)
      this.notes = newNotes
      this.addNote = false
    },

    handleCancelSave() {
      this.addNote = false
    },

    async deleteNote(id) {
      if (window.confirm('Are you sure?')) {
        this.deletingId = id
        const newNotes = this.notes.filter(note => note.id !== id)
        await updateNotesFile(newNotes)
        this.notes = newNotes
        this.deletingId = null
      }
    },

    async handleEditNote(note) {
      await this.handleSaveNote(note, true)
      this.handleCancelEditNote(note)
    },

    handleCancelEditNote(note) {
      this.editingIds = this.editingIds.filter(id => id !== note.id)
    },

    toHTML(content) {
      return marked(content)
    },

    async importOldNotes(notes) {
      if (this.importingOldNotes) {
        return
      }
      this.importingOldNotes = true
      const oldNotes = await Promise.all(
        notes.map(async note => {
          const file = await userSession.getFile(`notes/${note.id}.json`)
          await userSession.deleteFile(`notes/${note.id}.json`)
          return Object.assign(note, JSON.parse(file))
        })
      )
      const newNotes = this.notes.concat(oldNotes)
      await updateNotesFile(newNotes, { disableOldNotes: true })
      this.notes = newNotes
      this.oldNotes = null

      // Remove old notes.json
      await userSession.deleteFile('notes.json')
    }
  },

  computed: {
    sortedNotes() {
      return (
        this.notes &&
        this.notes.sort((a, b) => {
          const aDate = new Date(a.createdAt)
          const bDate = new Date(b.createdAt)
          return bDate - aDate
        })
      )
    }
  }
})
</script>

<style scoped>
.add-note-trigger {
  border: 1px solid var(--border-color);
  padding: 20px;
  border-radius: var(--card-border-radius);
  cursor: text;
  background: var(--card-bg);
  display: flex;
  align-items: center;

  & svg {
    margin-right: 10px;
  }
}

.add-note {
  border: 1px solid var(--border-color);
  padding: 20px;
  border-radius: var(--card-border-radius);
  width: 100%;
  font-size: 1rem;
  background: var(--card-bg);
}

.notes {
}

.note {
  border: 1px solid var(--border-color);
  border-radius: var(--card-border-radius);
  padding: 15px 20px;
  background: var(--card-bg);
  margin-bottom: 20px;
}

.note-title {
  margin-bottom: 10px;
}

.note-meta {
  font-size: 0.875rem;
  margin-top: 10px;
  color: var(--fade-text-color);
}

.note-meta {
  & span {
    margin-right: 20px;
  }

  & .note-action:hover {
    cursor: pointer;
    color: var(--active-text-color);
  }
}

.sticky {
  position: sticky;
  top: 15px;
}

.editing {
  margin-bottom: 15px;
  font-size: 0.875rem;
  display: flex;
  align-items: center;
  cursor: pointer;

  & svg {
    margin-right: 5px;
  }

  &:hover {
    color: var(--active-text-color);
  }
}

.links {
  margin-top: 15px;
  font-size: 0;

  & span {
    font-size: 0.7rem;
    &:not(:last-child) {
      &:after {
        content: '•';
        margin: 0 8px;
        opacity: 0.5;
      }
    }
  }

  & a {
    text-decoration: none;
    &:hover {
      text-decoration: underline;
    }
  }
}
</style>
