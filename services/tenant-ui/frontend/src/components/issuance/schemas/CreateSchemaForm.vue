<template>
  <form @submit.prevent="handleSubmit(!v$.$invalid)">
    <div class="container">
      <div v-if="isCopy" style="text-align: center">
        <div class="info-box">
          {{ $t('configuration.schemas.copyMessage') }}
        </div>
        <p style="font-weight: bold">{{ selectedSchema?.schema_id }}</p>
      </div>
      <div class="mt-2">
        <!-- schema name -->
        <ValidatedField
          :placeholder="isCopy ? selectedSchema?.schema?.name : ''"
          :field-name="'name'"
          :label="$t('issue.schemaName')"
          :loading="loading"
          :submitted="submitted"
          :validation="v$"
          :advanced-is-error="isError"
        />
        <!-- schema version -->
        <ValidatedField
          :placeholder="isCopy ? selectedSchema?.schema?.version : ''"
          :field-name="'version'"
          :label="$t('issue.schemaVersion')"
          :loading="loading"
          :submitted="submitted"
          :validation="v$"
          :advanced-is-error="isError"
        />
        <!-- attributes -->
        <Attributes ref="attributes" :initial-attributes="initialAttributes" />
        <Button
          type="submit"
          :label="t('configuration.schemas.create')"
          class="mt-5 w-full"
          :disabled="formFields.name === '' && formFields.version === ''"
          :loading="loading"
        />
      </div>
    </div>
  </form>
</template>

<script setup lang="ts">
// Libraries
import { useVuelidate } from '@vuelidate/core';
import { helpers, required } from '@vuelidate/validators';
import { storeToRefs } from 'pinia';
import Button from 'primevue/button';
import { reactive, ref } from 'vue';
import { useI18n } from 'vue-i18n';
import { useToast } from 'vue-toastification';
// Source
import ValidatedField from '@/components/common/ValidatedField.vue';
import errorHandler from '@/helpers/errorHandler';
import { useGovernanceStore } from '@/store';
import { Attribute } from '@/types';
import { SchemaSendRequest } from '@/types/acapyApi/acapyInterface';
import Attributes from './Attributes.vue';

const toast = useToast();
const { t } = useI18n();

const governanceStore = useGovernanceStore();
const { loading, selectedSchema } = storeToRefs(useGovernanceStore());

const emit = defineEmits(['closed', 'success']);
const attributes = ref();

const props = defineProps({
  isCopy: {
    type: Boolean,
    required: false,
    default: false,
  },
  initialAttributes: {
    type: Array<Attribute>,
    required: false,
    default: [],
  },
  // This can be used if emit won't work from parent component
  onClose: {
    type: Function,
    required: false,
    default: () => {},
  },
});

// Form / Validation setup
const formFields = reactive({
  name: '',
  version: '',
});

let rules: { [key: string]: any } = {};

const mustBeDecimal = (value: string) => /^\d+(\.\d+)(\.\d+)?$/.test(value);
rules = {
  name: { required },
  version: {
    required,
    mustBeDecimal: helpers.withMessage(
      t('configuration.schemas.mustBeDecimal'),
      mustBeDecimal
    ),
  },
};

const changedField = (field: string) => {
  if (field === 'name' || field === 'version')
    if (formFields[field] !== selectedSchema.value?.schema?.[field])
      return true;
  return false;
};

const isError = (v: any, field: string) => {
  if (props.isCopy)
    return v[field].$error && submitted.value && !changedField(field);
  return v[field].$error && submitted.value;
};

// Add copy rules if isCopy
if (props.isCopy) {
  const validCopy = () => {
    if (
      (changedField('name') && formFields.name !== '') ||
      (changedField('version') && formFields.version !== '')
    )
      return true;
    return false;
  };

  const setRule = (value: string | undefined, field: string) => {
    if (value)
      rules[field] = {
        ...rules[field],
        validCopy: helpers.withMessage(
          t('configuration.schemas.invalidCopy', {
            field,
            value,
          }),
          validCopy
        ),
      };
  };
  if (selectedSchema.value) {
    setRule(selectedSchema.value.schema?.name, 'name');
    setRule(selectedSchema.value.schema?.version, 'version');
  }

  delete rules.name.required;
  delete rules.version.required;
}

const v$ = useVuelidate(rules, formFields);

// Form submission
const submitted = ref(false);
const handleSubmit = async (isFormValid: boolean) => {
  submitted.value = true;

  if (!isFormValid) {
    return;
  }
  try {
    const attributeNames = attributes.value?.attributes
      .filter((x: Attribute) => x.name !== '')
      .map((attribute: Attribute) => attribute.name);

    if (!attributeNames.length) {
      toast.error(t('configuration.schemas.emptyAttributes'));
      return;
    }

    const payload: SchemaSendRequest = {
      attributes: attributeNames,
      schema_name: formFields.name || selectedSchema.value?.schema?.name || '',
      schema_version:
        formFields.version || selectedSchema.value?.schema?.version || '',
    };

    await governanceStore.createSchema(payload);

    toast.success(t('configuration.schemas.postStart'));
    emit('success');
    emit('closed', payload);
    if (props.onClose) props.onClose(payload);
  } catch (error) {
    errorHandler({
      error,
      existsMessage: t('configuration.schemas.alreadyExists'),
    });
  } finally {
    submitted.value = false;
  }
};
</script>

<style scoped>
.container {
  min-width: 450px;
  display: inline-block;
  text-align: left;
}
form {
  display: block;
  text-align: center;
}
.info-box {
  background-color: #eff6ff;
  border-radius: 5px;
  padding: 10px;
  margin-bottom: 10px;
}
</style>
