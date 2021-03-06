<template>
  <div>
    <div v-for="(f, k) in financialDocuments" :key="k">
    <ValidationObserver v-slot="{ invalid, validate }">
      <form name="form" @submit.prevent="validate().then(save(f))">
      <div class="rf-grid-row rf-mb-3w" style="justify-content: space-between">
        <span> Revenu {{ k + 1 }} </span>
        <DfButton
          class="rf-btn"
          size="small"
          @on-click="removeFinancial(f)"
        >
          {{$t('delete-financial')}}
        </DfButton>
      </div>
      <div>
        <label class="rf-label" for="select">
          Attention, Veuillez renseigner uniquement vos propres revenus.
        </label>
        <select
          v-model="f.documentType"
          class="rf-select rf-mb-3w"
          id="select"
          name="select"
        >
          <option v-for="d in documents" :value="d" :key="d.key">
            {{ $t(d.key) }}
          </option>
        </select>
      </div>
      <div v-if="f.documentType && f.documentType.key">
        <div>
          <validation-provider
            :rules="{ required:true, regex: /^[0-9., ]+$/ }"
            v-slot="{ errors }"
          >
            <div
              class="rf-input-group"
              :class="errors[0] ? 'rf-input-group--error' : ''"
            >
              <label for="monthlySum" class="rf-label"
                >{{ $t("monthlySum-label") }} :</label
              >
              <input
                id="monthlySum"
                :placeholder="$t('monthlySum')"
                type="text"
                v-model="f.monthlySum"
                name="monthlySum"
                class="validate-required form-control rf-input"
                required
              />
              <span class="rf-error-text" v-if="errors[0]">{{
                $t(errors[0])
              }}</span>
              <span class="rf-error-text" v-if="f.monthlySum > 10000">
                {{ $t("high-salary") }}
              </span>
              <span class="rf-error-text" v-if="f.monthlySum <= 0">
                {{ $t("low-salary") }}
              </span>
            </div>
          </validation-provider>
        </div>
        <div class="rf-mb-3w">
          {{ f.documentType.explanationText }}
        </div>
        <div class="rf-mb-3w">
          <FileUpload
            :current-status="f.fileUploadStatus"
            @add-files="addFiles(f, ...arguments)"
            @reset-files="resetFiles(f, ...arguments)"
          ></FileUpload>
        </div>
        <div class="rf-col-12 rf-mb-3w">
          <input
            type="checkbox"
            :id="`noDocument${k}`"
            value="false"
            v-model="f.noDocument"
          />
          <label :for="`noDocument${k}`">
            <template v-if="f.documentType.key === 'salary'">{{
              $t("noDocument-salary")
            }}</template>
            <template v-if="f.documentType.key === 'pension'">{{
              $t("noDocument-pension")
            }}</template>
            <template v-if="f.documentType.key === 'rent'">{{
              $t("noDocument-rent")
            }}</template>
            <template v-if="f.documentType.key === 'trading'">{{
              $t("noDocument-trading")
            }}</template>
            <template v-if="f.documentType.key === 'social-service'">{{
              $t("noDocument-social")
            }}</template>
          </label>
        </div>
        <div class="rf-mb-5w" v-if="f.noDocument">
          <div class="rf-input-group">
            <label class="rf-label" :for="`customText${k}`">{{
              $t("customText")
            }}</label>
            <input
              v-model="f.customText"
              class="form-control rf-input validate-required"
              :id="`customText${k}`"
              name="customText"
              placeholder=""
              type="text"
            />
          </div>
        </div>
      </div>
      <div
        v-if="financialFiles(f).length > 0"
        class="rf-col-lg-8 rf-col-md-12 rf-mb-3w"
      >
        <ListItem
          v-for="(file, k) in financialFiles(f)"
          :key="k"
          :file="file"
          @remove="remove(f, file)"
        />
      </div>
      <div class="rf-col-12 rf-mb-5w" v-if="f.documentType">
        <button
          class="rf-btn"
          type="submit"
          :disabled="f.files.length <= 0 && !f.noDocument"
        >
          Enregistrer la pièce
        </button>
      </div>
      <div class="rf-mb-5w">
        <DocumentInsert
          :allow-list="f.documentType.acceptedProofs"
          :block-list="f.documentType.refusedProofs"
        ></DocumentInsert>
      </div>
      </form>
    </ValidationObserver>
      <hr />
    </div>
    <div class="rf-col-12 rf-mb-5w">
      <button class="rf-btn" type="submit" @click="addFinancial()">
        Ajouter un revenu
      </button>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from "vue-property-decorator";
import { DocumentType } from "df-shared/src/models/Document";
import DocumentInsert from "@/components/documents/DocumentInsert.vue";
import FileUpload from "@/components/uploads/FileUpload.vue";
import { mapGetters } from "vuex";
import { UploadStatus } from "../uploads/UploadStatus";
import ListItem from "@/components/uploads/ListItem.vue";
import { User } from "df-shared/src/models/User";
import { DfFile } from "df-shared/src/models/DfFile";
import { DfDocument } from "df-shared/src/models/DfDocument";
import { Guarantor } from "df-shared/src/models/Guarantor";
import { extend } from "vee-validate";
import { RegisterService } from "../../services/RegisterService";
import DfButton from "df-shared/src/Button/Button.vue";
import { ValidationObserver, ValidationProvider } from "vee-validate";
import { required, regex } from "vee-validate/dist/rules";

extend("regex", {
  ...regex,
  message: "number-not-valid"
});

extend("required", {
  ...required,
  message: "field-required"
});

class F {
  id?: number;
  documentType = new DocumentType();
  noDocument = false;
  files: DfFile[] = [];
  fileUploadStatus = UploadStatus.STATUS_INITIAL;
  customText = "";
  monthlySum?: number;
}

@Component({
  components: {
    ValidationProvider,
    ValidationObserver,
    DocumentInsert,
    FileUpload,
    ListItem,
    DfButton
  },
  computed: {
    ...mapGetters({
      user: "userToEdit"
    })
  }
})
export default class Financial extends Vue {
  MAX_FILE_COUNT = 5;
  MAX_FILE_SIZE = 5;

  user!: User | Guarantor;
  financialDocuments: F[] = [];

  mounted() {
    this.initialize();
  }

  initialize() {
    this.financialDocuments = [];
    if (this.user.documents !== null) {
      const docs = this.user.documents?.filter((d: DfDocument) => {
        return d.documentCategory === "FINANCIAL";
      });
      if (docs !== undefined && docs.length > 0) {
        docs.forEach((d: DfDocument) => {
          const f = new F();
          f.noDocument = d.noDocument || false;
          f.customText = d.customText || "";
          f.monthlySum = d.monthlySum || 0;
          f.id = d.id;

          const localDoc = this.documents.find((d2: DocumentType) => {
            return d2.value === d.documentSubCategory;
          });
          if (localDoc !== undefined) {
            f.documentType = localDoc;
          }
          this.financialDocuments.push(f);
        });
      }
    } else {
      this.financialDocuments.push(new F());
    }
  }

  addFiles(f: F, fileList: File[]) {
    const nf = Array.from(fileList).map(f => {
      return { name: f.name, file: f, size: f.size };
    });
    f.files = [...f.files, ...nf];
  }
  resetFiles(f: F) {
    f.fileUploadStatus = UploadStatus.STATUS_INITIAL;
  }
  save(f: F) {
    const fieldName = "documents";
    const formData = new FormData();
    if (!f.noDocument) {
      const newFiles = f.files.filter(f => {
        return !f.id;
      });
      if (!newFiles.length) return;

      if (
        f.documentType.maxFileCount &&
        this.financialFiles(f).length > f.documentType.maxFileCount
      ) {
        Vue.toasted.global.max_file();
        return;
      }

      Array.from(Array(newFiles.length).keys()).map(x => {
        const f: File = newFiles[x].file || new File([], "");
        formData.append(`${fieldName}[${x}]`, f, newFiles[x].name);
      });
    }

    const typeDocumentFinancial = f.documentType?.value || "";
    formData.append("typeDocumentFinancial", typeDocumentFinancial);
    formData.append("noDocument", f.noDocument ? "true" : "false");
    if (f.monthlySum) {
      formData.append("monthlySum", f.monthlySum.toString());
    }
    if (f.id) {
      formData.append("id", f.id.toString());
    }
    formData.append("customText", f.customText);

    f.fileUploadStatus = UploadStatus.STATUS_SAVING;
    if (this.$store.getters.isGuarantor && this.$store.getters.guarantor.id) {
      formData.append("guarantorId", this.$store.getters.guarantor.id);
    }
    const loader = this.$loading.show();
    RegisterService.saveFinancial(formData)
      .then(() => {
        f.files = [];
        f.fileUploadStatus = UploadStatus.STATUS_INITIAL;
        Vue.toasted.global.save_success();
      })
      .catch(() => {
        f.fileUploadStatus = UploadStatus.STATUS_FAILED;
        Vue.toasted.global.save_failed();
      })
      .finally(() => {
        this.$store.dispatch("loadUser").then(() => {
          this.initialize();
        });
        loader.hide();
      });
  }

  financialFiles(f: F) {
    const newFiles = f.files.map((file: DfFile) => {
      return {
        documentSubCategory: f.documentType?.value,
        id: file.name,
        name: file.name,
        size: file.size
      };
    });
    const existingFiles =
      this.$store.getters.getDocuments?.find((d: DfDocument) => {
        return d.id === f.id;
      })?.files || [];
    return [...newFiles, ...existingFiles];
  }

  remove(f: F, file: DfFile) {
    if (file.path && file.id) {
      RegisterService.deleteFile(file.id);
    } else {
      f.files = f.files.filter((f: DfFile) => {
        return f.name !== file.name;
      });
    }
  }

  addFinancial() {
    this.financialDocuments.push(new F());
  }

  removeFinancial(f: DfDocument) {
    if (f.id) {
      const loader = Vue.$loading.show();
      this.$store.dispatch("deleteDocument", f.id).then(null, () => {
        Vue.toasted.global.error();
      }).finally(() => {
        loader.hide();
        this.initialize();
      })
    } else {
      this.financialDocuments = this.financialDocuments.filter((d:DfDocument) => { return d !== f});
    }
  }

  documents: DocumentType[] = [
    {
      key: "salary",
      value: "SALARY",
      explanationText:
        "J’ajoute mes trois derniers bulletins de salaire, ou un justificatif de mes indemnités de stage, ou mes deux derniers bilans comptables (non-salariés).",
      acceptedProofs: [
        "3 derniers bulletins de salaire",
        "Justificatif de versement des indemnités de stage",
        "2 derniers bilans comptables ou, si nécessaire, attestation des ressources pour l’exercice en cours délivrés par un comptable (non-salariés)"
      ],
      refusedProofs: [
        "Pièces trop anciennes",
        "Attestation de l’employeur",
        "Relevés de comptes bancaires",
        "RIB",
        "Avis d’imposition"
      ],
      maxFileCount: 3
    },
    {
      key: "social-service",
      value: "SOCIAL_SERVICE",
      explanationText:
        "J’ajoute mes trois derniers justificatifs de versement de prestations sociales (ARE, CAF, Crous…), un justificatif d’ouverture des droits, ou une attestation de simulation pour les aides au logement.",
      acceptedProofs: [
        "3 derniers justificatifs de versement des prestations sociales et familiales et allocations (ARE, CAF, Crous, etc.)",
        "Justificatif de l’ouverture des droits établis par l’organisme payeur",
        "Attestation de simulation pour les aides au logement établie par la CAF ou par la MSA pour le locataire"
      ],
      refusedProofs: [
        "Pièces trop anciennes",
        "Relevés de comptes bancaires",
        "RIB",
        "Avis d’imposition"
      ],
      maxFileCount: 3
    },
    {
      key: "rent",
      value: "RENT",
      explanationText:
        "J’ajoute un justificatif de paiement de rente, ou un avis d’imposition avec noms et revenus de la rente visibles.",
      acceptedProofs: [
        "Justification de revenus fonciers, de rentes viagères ou de revenus de valeurs et capitaux mobiliers",
        "Titre de propriété d’un bien immobilier ou dernier avis de taxe foncière",
        "Dernier ou avant-dernier avis d’imposition avec nom et revenus de la rente visibles"
      ],
      refusedProofs: ["Relevés de comptes bancaires", "RIB"],
      maxFileCount: 3
    },
    {
      key: "pension",
      value: "PENSION",
      explanationText:
        "J’ajoute un bulletin de pension, une attestation de pension, ou un avis d’imposition avec noms et revenus de la pension visibles.",
      acceptedProofs: [
        "Justificatif de versement des indemnités, retraites, pensions perçues lors des 3 derniers mois ou justificatif de l’ouverture des droits établis par l’organisme payeur",
        "Dernier ou avant-dernier avis d’imposition avec nom et revenus de la pension visibles"
      ],
      refusedProofs: [
        "Pièces trop anciennes",
        "Relevés de comptes bancaires",
        "RIB"
      ],
      maxFileCount: 3
    },
    {
      key: "trading",
      value: "TRADING",
      explanationText: "J’ajoute un avis d’attribution de bourse.",
      acceptedProofs: ["Avis d’attribution de bourse pour l’année en cours"],
      refusedProofs: [
        "Pièces trop anciennes",
        "Relevés de comptes bancaires",
        "RIB"
      ],
      maxFileCount: 3
    }
  ];
}
</script>

<style scoped lang="scss"></style>

<i18n>
{
"en": {
"salary": "Salary",
"social-service": "Social benefit payments",
"rent": "Annuities",
"pension": "Pensions",
"trading": "Trading",
"monthlySum": "Value in euros",
"monthlySum-label": "Salary (after tax)",
"noDocument-social": "I cannot provide proof of payment of social benefits",
"noDocument-salary": "I cannot provide my last three payslips",
"noDocument-rent": "I cannot provide proof of rent",
"noDocument-pension": "I cannot provide proof of pension",
"noDocument-trading": "I cannot provide proof of trading",
"customText": "In order to improve my file, I explain why I cannot provide my last three payslips:",
"high-salary": "You have entered a salary greater than € 10,000 are you sure you have entered your monthly salary?",
"low-salary": "You have entered a salary equal to 0 € are you sure you have entered your monthly salary?",
"number-not-valid": "Number not valid",
"delete-financial":  "Delete this salary",
"field-required": "This field is required"
},
"fr": {
"salary": "Salaire",
"social-service": "Versement de prestations sociales",
"rent": "Rentes",
"pension": "Pensions",
"trading": "Bourses",
"monthlySum": "Montant en euros",
"monthlySum-label": "Montant du revenu (après impôts)",
"noDocument-social": "Je ne peux pas fournir de justificatifs de versement de prestations sociales",
"noDocument-salary": "Je ne peux pas fournir mes trois derniers bulletins de salaire",
"noDocument-pension": "Je ne peux pas fournir de justificatifs de versement de pension",
"noDocument-rent": "Je ne peux pas fournir de justificatifs de versement de rente",
"noDocument-trading": "Je ne peux pas fournir de justificatifs d'attribution de bourse",
"customText": "Afin d'améliorer mon dossier, j'explique pourquoi je ne peux pas fournir mes trois derniers bulletins de salaire :",
"high-salary": "Vous avez saisi un salaire supérieur à 10 000€ êtes-vous sûr d'avoir saisi votre salaire mensuel ?",
"low-salary": "Vous avez saisi un salaire égal à 0€ êtes-vous sûr d'avoir saisi votre salaire mensuel ?",
"number-not-valid": "Nombre incorrect",
"delete-financial":  "Supprimer ce revenu",
"field-required": "Ce champ est requis"
}
}
</i18n>
