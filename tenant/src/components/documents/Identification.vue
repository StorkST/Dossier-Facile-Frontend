<template>
  <div>
    <div v-if="isGuarantor()" class="rf-grid-row rf-grid-row--center">
      <div class="rf-col-12 rf-mb-3w">
        <validation-provider rules="required" v-slot="{ errors }">
          <div
            class="rf-input-group"
            :class="errors[0] ? 'rf-input-group--error' : ''"
          >
            <label class="rf-label" for="lastname"
              >{{ $t("lastname") }} :</label
            >
            <input
              v-model="lastName"
              class="form-control rf-input validate-required"
              id="lastname"
              name="lastname"
              :placeholder="$t('lastname')"
              type="text"
            />
            <span class="rf-error-text" v-if="errors[0]">{{ errors[0] }}</span>
          </div>
        </validation-provider>
      </div>
      <div class="rf-col-12 rf-mb-3w">
        <validation-provider rules="required" v-slot="{ errors }">
          <div
            class="rf-input-group"
            :class="errors[0] ? 'rf-input-group--error' : ''"
          >
            <label for="firstname" class="rf-label"
              >{{ $t("firstname") }} :</label
            >
            <input
              id="firstname"
              :placeholder="$t('firstname')"
              type="text"
              v-model="firstName"
              name="firstname"
              class="validate-required form-control rf-input"
            />
            <span class="rf-error-text" v-if="errors[0]">{{
              $t(errors[0])
            }}</span>
          </div>
        </validation-provider>
      </div>
    </div>
    <div>
      <label class="rf-label" for="select">
        J’ajoute une pièce d’identité en cours de validité. Attention, veillez à
        ajouter votre pièce recto-verso !
      </label>
      <select
        v-model="identificationDocument"
        class="rf-select rf-mb-3w"
        id="select"
        name="select"
      >
        <option v-for="d in documents" :value="d" :key="d.key">
          {{ $t(d.key) }}
        </option>
      </select>
    </div>
    <div v-if="identificationDocument.key">
      <div v-if="identificationDocument.explanationText" class="rf-mb-3w">
        <p v-html="identificationDocument.explanationText"></p>
      </div>
      <div class="rf-mb-3w">
        <FileUpload
          :current-status="fileUploadStatus"
          @add-files="addFiles"
          @reset-files="resetFiles"
        ></FileUpload>
      </div>
    </div>
    <div
      v-if="identificationFiles().length > 0"
      class="rf-col-lg-8 rf-col-md-12 rf-mb-3w"
    >
      <ListItem
        v-for="(file, k) in identificationFiles()"
        :key="k"
        :file="file"
        @remove="remove(file)"
      />
    </div>
    <div class="rf-col-12 rf-mb-2w" v-if="identificationDocument">
      <button
        class="rf-btn"
        type="submit"
        @click="save"
        :disabled="files.length <= 0"
      >
        Enregistrer la pièce
      </button>
    </div>
    <div class="rf-mb-5w" v-if="identificationDocument.key">
      <DocumentInsert
        :allow-list="identificationDocument.acceptedProofs"
        :block-list="identificationDocument.refusedProofs"
      ></DocumentInsert>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Vue, Watch } from "vue-property-decorator";
import { mapGetters, mapState } from "vuex";
import DocumentInsert from "@/components/documents/DocumentInsert.vue";
import FileUpload from "@/components/uploads/FileUpload.vue";
import { DocumentType } from "df-shared/src/models/Document";
import { UploadStatus } from "../uploads/UploadStatus";
import ListItem from "@/components/uploads/ListItem.vue";
import { User } from "df-shared/src/models/User";
import { DfFile } from "df-shared/src/models/DfFile";
import { DfDocument } from "df-shared/src/models/DfDocument";
import { ValidationProvider } from "vee-validate";
import { Guarantor } from "df-shared/src/models/Guarantor";
import { RegisterService } from "../../services/RegisterService";

@Component({
  components: { DocumentInsert, FileUpload, ListItem, ValidationProvider },
  computed: {
    ...mapState({
      selectedGuarantor: "selectedGuarantor"
    }),
    ...mapGetters({
      user: "userToEdit"
    })
  }
})
export default class Identification extends Vue {
  MAX_FILE_COUNT = 3;

  user!: User | Guarantor;
  selectedGuarantor!: Guarantor;
  fileUploadStatus = UploadStatus.STATUS_INITIAL;
  files: DfFile[] = [];
  uploadProgress: {
    [key: string]: { state: string; percentage: number };
  } = {};
  identificationDocument = new DocumentType();
  firstName = "";
  lastName = "";

  @Watch("selectedGuarantor")
  onGuarantorChange(val: Guarantor) {
    this.firstName = val.firstName || "";
    this.lastName = val.lastName || "";
  }

  mounted() {
    if (this.user.documents !== null) {
      const doc = this.user.documents?.find((d: DfDocument) => {
        return d.documentCategory === "IDENTIFICATION";
      });
      if (doc !== undefined) {
        const localDoc = this.documents.find((d: DocumentType) => {
          return d.value === doc.documentSubCategory;
        });
        if (localDoc !== undefined) {
          this.identificationDocument = localDoc;
        }
      }
    }

    if (this.selectedGuarantor.firstName) {
      this.firstName = this.selectedGuarantor.firstName;
    }
    if (this.selectedGuarantor.lastName) {
      this.lastName = this.selectedGuarantor.lastName;
    }
  }

  addFiles(fileList: File[]) {
    const nf = Array.from(fileList).map(f => {
      return { name: f.name, file: f, size: f.size };
    });
    this.files = [...this.files, ...nf];
  }

  resetFiles() {
    this.fileUploadStatus = UploadStatus.STATUS_INITIAL;
  }

  save() {
    this.uploadProgress = {};
    const fieldName = "documents";
    const formData = new FormData();
    const newFiles = this.files.filter(f => {
      return !f.id;
    });
    if (!newFiles.length) return;

    if (
      this.identificationDocument.maxFileCount &&
      this.identificationFiles().length >
        this.identificationDocument.maxFileCount
    ) {
      Vue.toasted.global.max_file();
      return;
    }

    Array.from(Array(newFiles.length).keys()).map(x => {
      const f: File = newFiles[x].file || new File([], "");
      formData.append(`${fieldName}[${x}]`, f, newFiles[x].name);
    });

    formData.append(
      "typeDocumentIdentification",
      this.identificationDocument.value
    );

    this.fileUploadStatus = UploadStatus.STATUS_SAVING;
    if (this.$store.getters.isGuarantor) {
      formData.append("firstName", this.firstName);
      formData.append("lastName", this.lastName);
      if (this.$store.getters.guarantor.id) {
        formData.append("guarantorId", this.$store.getters.guarantor.id);
      }
    }
    const loader = this.$loading.show();
    RegisterService.saveIdentification(formData)
      .then(() => {
        this.fileUploadStatus = UploadStatus.STATUS_INITIAL;
        this.files = [];
        Vue.toasted.global.save_success();
      })
      .catch(() => {
        this.fileUploadStatus = UploadStatus.STATUS_FAILED;
        Vue.toasted.global.save_failed();
      })
      .finally(() => {
        this.$store.dispatch("loadUser");
        loader.hide();
      });
  }

  identificationFiles() {
    const newFiles = this.files.map(f => {
      return {
        documentSubCategory: this.identificationDocument.value,
        id: f.name,
        name: f.name,
        file: f.file,
        size: f.file?.size
      };
    });
    const existingFiles =
      this.$store.getters.getDocuments?.find((d: DfDocument) => {
        return d.documentCategory === "IDENTIFICATION";
      })?.files || [];
    return [...newFiles, ...existingFiles];
  }

  remove(file: DfFile) {
    if (file.path && file.id) {
      RegisterService.deleteFile(file.id);
    } else {
      this.files = this.files.filter((f: DfFile) => {
        return f.name !== file.name;
      });
    }
  }

  isGuarantor() {
    return this.$store.getters.isGuarantor;
  }

  documents: DocumentType[] = [
    {
      key: "identity-card",
      value: "FRENCH_IDENTITY_CARD",
      explanationText: "Attention veillez à ajouter votre pièce <b>recto-verso !</b>",
      acceptedProofs: ["Carte d’identité française <b>recto-verso</b>"],
      refusedProofs: [
        "Carte d’identité <b>sans le verso ou périmée</b>",
        "Tout autre document"
      ],
      maxFileCount: 3
    },
    {
      key: "passport",
      value: "FRENCH_PASSPORT",
      acceptedProofs: ["Passport français (pages 2 et 3)"],
      refusedProofs: ["Tout autre document"],
      maxFileCount: 3
    },
    {
      key: "permit",
      value: "FRENCH_RESIDENCE_PERMIT",
      acceptedProofs: [
        "Carte de séjour en France temporaire recto-verso en cours de validité, ou périmée si elle est accompagnée du récépissé de la demande de renouvellement de carte de séjour",
        "Visa de travail ou d’études temporaire en France"
      ],
      refusedProofs: ["Tout autre document"],
      maxFileCount: 3
    },
    {
      key: "other",
      value: "OTHER_IDENTIFICATION",
      acceptedProofs: [
        "Carte d’identité étrangère <b>recto-verso</b>",
        "Passeport étranger (pages 2 et 3)",
        "Permis de conduire français ou étranger <b>recto-verso</b>",
        "Carte de résident",
        "Carte de ressortissant d’un État membre de l’UE ou de l’EEE"
      ],
      refusedProofs: ["Tout autre document"],
      maxFileCount: 3
    }
  ];
}
</script>

<style scoped lang="scss">
table {
  border-collapse: collapse;
}

table,
th,
td {
  border: 1px solid #ececec;
}
</style>

<i18n>
{
"en": {
  "identity-card": "Carte nationale d’identité",
  "passport": "Passeport",
  "permit": "Permis de conduire",
  "other": "Autre",
  "files": "Documents",
  "lastname": "Lastname",
  "firstname": "Firstname"
},
"fr": {
  "identity-card": "Carte nationale d’identité",
  "passport": "Passeport",
  "permit": "Permis de conduire",
  "other": "Autre",
  "files": "Documents",
  "lastname": "Nom",
  "firstname": "Prénom"
}
}
</i18n>
