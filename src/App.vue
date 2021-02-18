<template>
  <v-app>
    <v-app-bar app flat color="primary" dark>
      <div class="d-flex align-center display-1 font-weight-bold">
        Kindle 2 PDF
      </div>
      <v-spacer />
      <v-btn href="https://twitter.com/@dakmig" depressed color="blue"
        >Follow on Twitter</v-btn
      >
    </v-app-bar>

    <v-main>
      <v-snackbar
        v-model="snack.show"
        :timeout="snack.timeout"
        :color="snack.color"
        shaped
        top
        left
      >
        {{ snack.message }}
        <template v-slot:action="{ attrs }">
          <v-btn text v-bind="attrs" @click="snack.show = false"> Close </v-btn>
        </template>
      </v-snackbar>
      <v-dialog
        v-model="dialog"
        fullscreen
        hide-overlay
        transition="dialog-bottom-transition"
      >
        <v-card>
          <v-toolbar dark color="primary">
            <v-btn icon dark @click="dialog = false">
              <v-icon>mdi-close</v-icon>
            </v-btn>
            <v-toolbar-title>Generate PDF</v-toolbar-title>
            <v-spacer></v-spacer>
            <v-toolbar-items>
              <v-btn dark text @click="savePDF"> Save PDF </v-btn>
            </v-toolbar-items>
          </v-toolbar>
          <v-card-text ref="pdfArea">
            <div v-for="book in books" :key="book.title" class="mx-4">
              <h2 class="book-heading font-weight-bold mt-4 black--text">
                {{ book.title }}
              </h2>
              <p
                class="book-p my-4"
                v-for="(value, idx) in book.values"
                :key="idx"
              >
                <strong class="black--text">{{ idx + 1 }}</strong
                >. {{ value }}
              </p>
            </div>
          </v-card-text>
        </v-card>
      </v-dialog>

      <v-container
        class="d-flex flex-column justify-center align-center fill-height"
      >
        <h2 class="mb-6 font-weight-bold">
          The easiest and most secure way to convert your kindle highlights to
          PDF!
        </h2>
        <v-sheet
          color="white"
          elevation="2"
          height="400"
          width="500"
          class="pa-2"
          shaped
        >
          <file-upload
            :drop="true"
            :drop-directory="false"
            ref="upload"
            v-model="files"
            @input-filter="inputFilter"
            class="fill-height fill-width d-flex align-center justify-center font-weight-bold title-1 flex-column"
            :disabled="files.length > 0"
          >
            <v-icon
              :color="files.length === 0 ? 'primary' : 'success'"
              x-large
              class="mb-4"
              >mdi-file-upload</v-icon
            >

            <p v-if="files.length === 0">
              Drop or Click to upload clippings.txt
            </p>
            <div v-else>
              <p>Uploaded {{ files[0].file.name }} Successfully!</p>
            </div>
          </file-upload>
        </v-sheet>
        <div>
          <v-btn
            class="mt-4 mr-4"
            color="success"
            :disabled="files.length === 0"
            @click="parseKindle"
            >Convert</v-btn
          >
          <v-btn
            class="mt-4"
            color="error"
            @click="$refs.upload.clear()"
            :disabled="files.length === 0"
            >Clear</v-btn
          >
        </div>
      </v-container>
    </v-main>
    <v-footer padless>
      <v-col class="text-center" cols="12">
        {{ new Date().getFullYear() }} — Built by
        <strong> <a href="https://twitter.com/@dakmig">Daksh</a> </strong>,
        Creator of <strong><a href="https://nasch.io">Nasch</a></strong>
      </v-col>
    </v-footer>
  </v-app>
</template>

<script>
const END_OF_LINE = "\n";
const KINDLE_NOTE_DELIMITER = "==========";

// rewrote the parser orignally by https://elvisciotti.github.io/kindleparser/
function parseMyClippings(inputContent) {
  // read into map of sets
  let organisedNotesMap = new Map();
  inputContent.split(KINDLE_NOTE_DELIMITER).forEach((note) => {
    let lines = note.trim().split(END_OF_LINE);
    let title = lines[0].trim();
    if (title.length > 1) {
      let body = lines.slice(2).join(END_OF_LINE);
      if (!organisedNotesMap.has(title)) {
        organisedNotesMap.set(title, new Set());
      }
      let bodyTrimmed = trimNewLinesSpacesDots(body);
      bodyTrimmed = bodyTrimmed.charAt(0).toUpperCase() + bodyTrimmed.slice(1);
      organisedNotesMap.get(title).add(bodyTrimmed);
    }
  });

  const notesArray = Array.from(organisedNotesMap);
  const books = [];
  for (let i = 0; i < notesArray.length; i++) {
    const [title, texts] = notesArray[i];

    const temp = Array.from(Array.from(texts).values());
    const newNotes = new Set();

    temp.forEach((item) => {
      let highest = item;
      temp.forEach((item2) => {
        if (item2.indexOf(highest) > -1) {
          if (highest.length < item2.length) {
            highest = item2;
          }
        }
      });
      newNotes.add(highest);
    });

    books.push({ title, values: Array.from(newNotes.values()) });
  }
  return books;
}

function trimNewLinesSpacesDots(content) {
  return (
    content
      // starting
      .replace(/([\.\r\n]+)$/, "")
      .replace(/([\.\n]+)$/, "")
      // ending
      .replace(/(^[\.\r\n]+)/, "")
      .replace(/(^[\.\n]+)/, "")
      .trim()
  );
}
export default {
  name: "App",

  components: {},

  data: () => ({
    dialog: false,
    files: [],
    snack: {
      show: false,
      timeout: -1,
      color: "",
      message: "",
    },
    books: [],
  }),
  methods: {
    savePDF() {
      const el = this.$refs.pdfArea;
      this.openSnack("Saving PDF. Please wait!", "success", 5000);
      try {
        window
          .html2pdf()
          .from(el)
          .set({
            margin: 1,
            html2canvas: { scale: 1 },
            pagebreak: { mode: ["avoid-all", "css", "legacy"] },

            jsPDF: { unit: "in", format: "letter", orientation: "portrait" },
            filename: "kindle-clips.pdf",
          })
          .save();
      } catch (e) {
        console.log(e);
        this.openSnack(
          "Sorry failed to save! Please try again!",
          "error",
          5000
        );
      }
    },
    async parseKindle() {
      const file = this.files[0].file;
      const text = await file.text();

      this.books = parseMyClippings(text).sort(
        (a, b) => b.values.length - a.values.length
      );
      this.dialog = true;
    },
    openSnack(message, color = "primary", timeout = -1) {
      this.snack.message = message;
      this.snack.timeout = timeout;
      this.snack.color = color;
      this.snack.show = true;
    },
    inputFilter(newFile, oldFile, prevent) {
      if (newFile && !oldFile) {
        // Filter non-txt file
        if (!/\.(txt)$/i.test(newFile.name)) {
          this.openSnack("Please upload a .txt file only!", "error");
          return prevent();
        }
      }

      // Create a blob field
      newFile.blob = "";
      let URL = window.URL || window.webkitURL;
      if (URL && URL.createObjectURL) {
        newFile.blob = URL.createObjectURL(newFile.file);
      }
    },
  },
  watch: {
    "files.length"(val) {
      if (val === 0) {
        this.openSnack("Successfully cleared the file!", "success", 10000);
      } else {
        this.openSnack("Successfully uploaded the file!", "success", 1000);
      }
    },
  },
};
</script>


<style>
.fill-width {
  width: 100% !important;
}
.title-1 {
  font-size: 1.5rem;
}

.book-heading {
  font-size: 1.8rem !important;
  line-height: 2.6rem;
  font-weight: 900;
}
.book-p {
  font-size: 1.3rem;
  font-weight: 500;
  line-height: 1.95rem;
  margin-bottom: 0.75rem;
}
</style>