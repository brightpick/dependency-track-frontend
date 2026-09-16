<template>
  <b-row class="expanded-row">
    <b-col sm="6">
      <b-form-group label="Uploaded license" label-for="uploaded-license">
        <b-input-group>
          <b-form-input
            id="uploaded-license"
            :value="uploadedLicense.value"
            class="form-control disabled"
            readonly
            trim
          />
          <!-- which slot the upload used: the policies judge an id, a name
               and an expression differently -->
          <b-input-group-append v-if="uploadedLicense.kind" is-text>{{
            uploadedLicense.kind
          }}</b-input-group-append>
        </b-input-group>
      </b-form-group>
      <!-- brightscan's second opinion, stamped on the component at upload as
           the brightscan:license:* properties; the badge is the verdict -->
      <b-form-group label="Resolver license" label-for="resolver-license">
        <b-input-group>
          <b-form-input
            id="resolver-license"
            :value="resolverLicense.value"
            class="form-control disabled"
            readonly
            trim
          />
          <b-input-group-append v-if="resolverLicense.verdict" is-text>{{
            resolverLicense.verdict
          }}</b-input-group-append>
        </b-input-group>
      </b-form-group>
      <b-form-group
        v-if="component.description"
        :label="this.$t('message.description')"
        label-for="component-description"
      >
        <b-form-textarea
          id="component-description"
          :value="component.description"
          rows="7"
          class="form-control disabled"
          readonly
          trim
        />
      </b-form-group>
      <b-form-group
        v-if="component.purl"
        label="Package URL"
        label-for="component-purl"
      >
        <b-form-input
          id="component-purl"
          :value="component.purl"
          class="form-control disabled"
          readonly
          trim
        />
      </b-form-group>
    </b-col>
    <b-col sm="6">
      <b-form-group label="Audit Trail" label-for="audit-trail">
        <b-form-textarea
          id="audit-trail"
          v-model="auditTrail"
          rows="7"
          class="form-control disabled"
          readonly
          trim
        />
      </b-form-group>
      <b-form-group label="Comment" label-for="audit-comment">
        <b-form-textarea
          id="audit-comment"
          v-model="comment"
          rows="3"
          class="form-control"
          :disabled="!canEdit"
          trim
        />
        <div class="pull-right">
          <b-button
            size="sm"
            variant="outline-primary"
            :disabled="!comment || !canEdit || saving"
            @click="addComment"
            ><span class="fa fa-comment-o"></span>
            {{ this.$t('message.add_comment') }}</b-button
          >
        </div>
      </b-form-group>
      <!-- An override targets exactly one of the component's license slots.
           The inputs hold a draft; the stored override changes only through
           "Set override" and "Clear override", so a stray selection, a
           half-typed value or a details edit never touches it. -->
      <b-form-group label="License override" label-for="override-kind">
        <b-form-select
          id="override-kind"
          v-model="overrideKind"
          :options="overrideKindOptions"
          :disabled="!canEdit"
        />
      </b-form-group>
      <b-form-group v-if="overrideKind === '' && storedKind">
        <b-button
          variant="outline-danger"
          size="sm"
          :disabled="!canEdit || saving"
          @click="clearOverride"
          >Clear override</b-button
        >
        <small class="form-text"
          >Restores the uploaded license on this component.</small
        >
      </b-form-group>
      <b-form-group
        v-if="overrideKind === 'license'"
        label-for="override-license"
      >
        <b-input-group>
          <b-form-select
            id="override-license"
            v-model="overrideLicense"
            :options="selectableLicenses"
            :disabled="!canEdit"
          />
          <b-input-group-append>
            <b-button
              variant="outline-primary"
              :disabled="!canEdit || !overrideValue || saving"
              @click="setOverride"
              >Set override</b-button
            >
            <b-input-group-text
              v-b-tooltip.hover
              :title="createPolicyHelp"
              class="px-2"
              ><i class="fa fa-info-circle"></i
            ></b-input-group-text>
            <b-button
              variant="outline-primary"
              :disabled="!canCreatePolicy || saving"
              @click="createPolicy"
              >Create policy</b-button
            >
          </b-input-group-append>
        </b-input-group>
        <small v-if="createdPolicyId" class="form-text">
          Policy created —
          <router-link :to="'/policy/componentPolicies/' + createdPolicyId"
            >edit it in Policy Management</router-link
          >
        </small>
      </b-form-group>
      <!-- a stored free-text name is shown as it is; it can only be replaced
           by a catalogue license or an expression, or cleared -->
      <b-form-group
        v-if="overrideKind === 'expression' || overrideKind === 'name'"
        label-for="override-text"
      >
        <b-input-group>
          <b-form-input
            id="override-text"
            v-model="overrideText"
            :placeholder="overrideTextPlaceholder"
            :disabled="!canEdit || overrideKind === 'name'"
            trim
            @keyup.enter="setOverride"
          />
          <b-input-group-append>
            <b-button
              variant="outline-primary"
              :disabled="!canEdit || !overrideValue || saving"
              @click="setOverride"
              >Set override</b-button
            >
            <b-input-group-text
              v-b-tooltip.hover
              :title="createPolicyHelp"
              class="px-2"
              ><i class="fa fa-info-circle"></i
            ></b-input-group-text>
            <b-button
              variant="outline-primary"
              :disabled="!canCreatePolicy || saving"
              @click="createPolicy"
              >Create policy</b-button
            >
          </b-input-group-append>
        </b-input-group>
        <small v-if="createdPolicyId" class="form-text">
          Policy created —
          <router-link :to="'/policy/componentPolicies/' + createdPolicyId"
            >edit it in Policy Management</router-link
          >
        </small>
      </b-form-group>
      <b-form-group label="Details" label-for="override-details">
        <b-form-textarea
          id="override-details"
          v-model="details"
          rows="3"
          class="form-control"
          :disabled="!canEdit"
          placeholder="Why this license override is correct (visible on the component)"
          trim
        />
        <div class="pull-right">
          <b-button
            size="sm"
            variant="outline-primary"
            :disabled="!canEdit || saving"
            @click="saveDetails"
            ><span class="fa fa-comment-o"></span>
            {{ this.$t('message.update_details') }}</b-button
          >
        </div>
      </b-form-group>
    </b-col>
  </b-row>
</template>

<script>
import common from '@/shared/common';
import permissionsMixin from '@/mixins/permissionsMixin';

// display label of a license slot, for the uploaded value and the snapshot
const KIND_LABELS = {
  id: 'SPDX id',
  expression: 'expression',
  name: 'name',
};

// the request field of each override slot; the other slots are sent null
const SLOT_FIELDS = {
  license: 'license',
  expression: 'license_expression',
  name: 'license_name',
};

export default {
  props: {
    component: Object,
    projectUuid: String,
    // the components table mounts the panel as its own Vue root, so a
    // callback stands in for an event: called after every saved change
    onChanged: Function,
  },
  mixins: [permissionsMixin],
  data() {
    return {
      selectableLicenses: [],
      // every override names catalogue licenses only — a license the
      // catalogue lacks is created on the License page first, then used here
      overrideKinds: [
        { value: '', text: '(no override)' },
        { value: 'license', text: 'License from the catalogue' },
        {
          value: 'expression',
          text: 'SPDX expression of catalogue licenses, e.g. MIT AND Apache-2.0',
        },
      ],
      properties: [],
      // the component as the API returns it after a save; the table row the
      // panel was opened with does not follow the saves
      liveComponent: null,
      analysisId: null,
      // the stored override: its slot ('' when the analysis holds none) and value
      storedKind: '',
      storedValue: null,
      declaredLicense: null,
      declaredLicenseKind: null,
      auditTrail: null,
      comment: null,
      // the draft in the inputs
      overrideKind: '',
      overrideLicense: null,
      overrideText: null,
      details: null,
      createdPolicyId: null,
      // a PUT is in flight: no second write may overlap it
      saving: false,
    };
  },
  computed: {
    canEdit: function () {
      // curation is its own authority, distinct from managing policies
      return this.isPermitted(this.PERMISSIONS.COMPONENT_ANALYSIS);
    },
    uploadedLicense: function () {
      // the snapshot the analysis keeps is the uploaded value whenever an
      // override was ever set — the component itself then carries the curated
      // value, or the restored one after a clear. Without a snapshot the
      // component is the source: the v2 list row (snake_case) the panel was
      // opened with, or the v1 object (camelCase) re-read after a save.
      if (this.declaredLicenseKind) {
        return {
          value: this.declaredLicense,
          kind: KIND_LABELS[this.declaredLicenseKind] || null,
        };
      }
      if (this.analysisId && this.storedKind) {
        return { value: '(none)', kind: null };
      }
      const c = this.liveComponent || this.component;
      const resolved = c.resolved_license || c.resolvedLicense;
      if (resolved) {
        return {
          value: resolved.license_id || resolved.licenseId || resolved.name,
          kind: KIND_LABELS.id,
        };
      }
      const expression = c.license_expression || c.licenseExpression;
      if (expression) {
        return { value: expression, kind: KIND_LABELS.expression };
      }
      if (c.license) {
        return { value: c.license, kind: KIND_LABELS.name };
      }
      return { value: '(none)', kind: null };
    },
    // the value the draft holds for the slot in use; a stored free-text
    // name is not a value to save again
    overrideValue: function () {
      if (this.overrideKind === 'license') {
        return this.overrideLicense || null;
      }
      if (this.overrideKind === 'expression') {
        return this.overrideText || null;
      }
      return null;
    },
    canCreatePolicy: function () {
      return Boolean(this.canEdit && this.component.purl && this.overrideValue);
    },
    createPolicyHelp: function () {
      return (
        'Creates a component policy for this package URL: in every project, ' +
        'each BOM import gives the matching components this license. A manual ' +
        'override on a component still wins. Refine the rule in Policy ' +
        'Management afterwards — e.g. drop the version to cover all versions.'
      );
    },
    overrideTextPlaceholder: function () {
      return 'MIT AND Apache-2.0 — every operand a license of the catalogue (create missing ones on the License page)';
    },
    overrideKindOptions: function () {
      // a stored free-text name is shown so it can be replaced, not offered anew
      if (this.storedKind !== 'name') {
        return this.overrideKinds;
      }
      return this.overrideKinds.concat([
        {
          value: 'name',
          text: 'Free-text name (stored — replace with a catalogue license or an expression)',
          disabled: true,
        },
      ]);
    },
    resolverLicense: function () {
      const prop = (name) => {
        const match = this.properties.find(
          (p) => p.groupName === 'brightscan' && p.propertyName === name,
        );
        return match ? match.propertyValue : null;
      };
      const verdict = prop('license:verdict');
      const said = prop('license:resolver');
      if (said) {
        return { value: said, verdict: verdict };
      }
      if (verdict === 'match') {
        // a match carries no resolver property: the resolver agreed with the
        // uploaded value on the canonical form, so that value is the answer
        return { value: this.uploadedLicense.value, verdict: verdict };
      }
      if (verdict === 'unresolved') {
        return {
          value: '(the resolver does not know this purl)',
          verdict: verdict,
        };
      }
      return { value: '(uploaded without resolver data)', verdict: verdict };
    },
  },
  created() {
    this.retrieveLicenses();
    this.loadProperties();
    this.loadAnalysis();
  },
  methods: {
    // the API's problem detail, else the given text, else the generic one
    failureText: function (error, fallback) {
      const response = error && error.response;
      const problem = response && response.data;
      const detail = problem && (problem.detail || problem.title);
      return detail || fallback || this.$t('condition.unsuccessful_action');
    },
    retrieveLicenses: function () {
      const url = `${this.$api.BASE_URL}/${this.$api.URL_LICENSE_CONCISE}`;
      this.axios
        .get(url)
        .then((response) => {
          // Empty option = no override; custom licenses have no licenseId,
          // their name doubles as the identifier.
          const licenses = [{ value: '', text: '' }];
          for (const license of response.data) {
            licenses.push({
              value: license.licenseId || license.name,
              text: license.name,
            });
          }
          this.selectableLicenses = licenses;
        })
        .catch((error) => {
          this.$toastr.w(this.failureText(error));
        });
    },
    loadProperties: function () {
      if (!this.component.uuid) {
        return;
      }
      const url = `${this.$api.BASE_URL}/${this.$api.URL_COMPONENT}/${this.component.uuid}/property`;
      this.axios
        .get(url)
        .then((response) => {
          this.properties = response.data || [];
        })
        .catch((error) => {
          this.$toastr.w(this.failureText(error));
        });
    },
    refreshComponent: function () {
      if (!this.component.uuid) {
        return Promise.resolve();
      }
      const url = `${this.$api.BASE_URL}/${this.$api.URL_COMPONENT}/${this.component.uuid}`;
      return this.axios
        .get(url)
        .then((response) => {
          this.liveComponent = response.data || null;
        })
        .catch((error) => {
          this.$toastr.w(this.failureText(error));
        });
    },
    identityMatches: function (analysis) {
      const norm = (value) => value || '';
      return (
        norm(analysis.purl) === norm(this.component.purl) &&
        norm(analysis.group) === norm(this.component.group) &&
        norm(analysis.name) === norm(this.component.name) &&
        norm(analysis.version) === norm(this.component.version)
      );
    },
    loadAnalysis: function () {
      const url = `${this.$api.BASE_URL}/api/v2/component-analyses?project=${this.projectUuid}`;
      return this.axios
        .get(url)
        .then((response) => {
          const analysis = (response.data.analyses || []).find(
            this.identityMatches,
          );
          if (!analysis) {
            this.analysisId = null;
            this.storedKind = '';
            this.storedValue = null;
            this.declaredLicense = null;
            this.declaredLicenseKind = null;
            this.auditTrail = null;
            return;
          }
          this.analysisId = analysis.id;
          this.storedKind = analysis.license
            ? 'license'
            : analysis.license_expression
              ? 'expression'
              : analysis.license_name
                ? 'name'
                : '';
          this.storedValue =
            analysis.license ||
            analysis.license_expression ||
            analysis.license_name ||
            null;
          // the draft starts from what is stored
          this.overrideKind = this.storedKind;
          this.overrideLicense = analysis.license || null;
          this.overrideText = analysis.license_expression || null;
          this.declaredLicense = analysis.declared_license;
          this.declaredLicenseKind = analysis.declared_license_kind;
          this.details = analysis.details;
          this.loadComments();
        })
        .catch((error) => {
          this.$toastr.w(this.failureText(error));
        });
    },
    loadComments: function () {
      if (!this.analysisId) {
        return;
      }
      const url = `${this.$api.BASE_URL}/api/v2/component-analyses/${this.analysisId}/comments`;
      this.axios
        .get(url)
        .then((response) => {
          let trail = '';
          for (const comment of response.data.comments || []) {
            if (comment.commenter) {
              trail += comment.commenter + ' - ';
            }
            // v2 serializes timestamps as epoch milliseconds
            trail += common.formatTimestamp(comment.timestamp, true);
            trail += '\n' + comment.comment + '\n\n';
          }
          this.auditTrail = trail;
        })
        .catch((error) => {
          this.$toastr.w(this.failureText(error));
        });
    },
    // one PUT of the analysis: the given slot (none when kind is null) and
    // the details
    saveAnalysis: function (kind, value, details, fallback) {
      if (this.saving) {
        return Promise.resolve();
      }
      this.saving = true;
      const url = `${this.$api.BASE_URL}/api/v2/component-analyses`;
      const body = {
        project_uuid: this.projectUuid,
        purl: this.component.purl || null,
        group: this.component.group || null,
        name: this.component.name,
        version: this.component.version || null,
        license: null,
        license_expression: null,
        license_name: null,
        details: details,
      };
      if (kind && value) {
        body[SLOT_FIELDS[kind]] = value;
      }
      return this.axios
        .put(url, body)
        .then(() => {
          this.$toastr.s(this.$t('message.updated'));
          return this.afterChange();
        })
        .catch((error) => {
          this.$toastr.w(
            this.failureText(
              error,
              error.response && error.response.status === 400 ? fallback : null,
            ),
          );
        })
        .finally(() => {
          this.saving = false;
        });
    },
    // a save changed the analysis and the component: re-read both, then let
    // the table refresh its override markers
    afterChange: function () {
      return Promise.all([this.loadAnalysis(), this.refreshComponent()])
        .then(() => (this.onChanged ? this.onChanged() : null))
        .catch((error) => {
          this.$toastr.w(this.failureText(error));
        });
    },
    setOverride: function () {
      // only a draft with a value is a save; a stored free-text name is
      // never saved again
      const kind = this.overrideKind;
      const value = this.overrideValue;
      if (!this.canEdit || !kind || kind === 'name' || !value) {
        return;
      }
      return this.saveAnalysis(
        kind,
        value,
        this.details || null,
        'Unknown license — create it as a custom license first',
      );
    },
    clearOverride: function () {
      if (!this.canEdit || !this.storedKind) {
        return;
      }
      return this.saveAnalysis(null, null, this.details || null);
    },
    saveDetails: function () {
      // the details ride the stored override, never the draft
      if (!this.canEdit) {
        return;
      }
      return this.saveAnalysis(
        this.storedKind,
        this.storedValue,
        this.details || null,
      );
    },
    createPolicy: function () {
      if (!this.canCreatePolicy) {
        return;
      }
      const purl = this.component.purl;
      const kind = this.overrideKind;
      const name = `curation ${purl}`;
      // a CEL string literal: escape what would end it
      const literal = purl.replace(/\\/g, '\\\\').replace(/"/g, '\\"');
      const url = `${this.$api.BASE_URL}/api/v2/component-policies`;
      this.axios
        .post(url, {
          name: name,
          description: `Created from the component audit of ${this.component.name}@${this.component.version || ''}`,
          condition: `component.purl == "${literal}"`,
          license: kind === 'license' ? this.overrideValue : null,
          license_expression: kind === 'expression' ? this.overrideValue : null,
          license_name: null,
          details: this.details || null,
          enabled: true,
          priority: 50,
        })
        .then((response) => {
          this.$toastr.s('Component policy created for ' + purl);
          const location = response.headers && response.headers.location;
          const match =
            location && /\/component-policies\/(\d+)$/.exec(location);
          if (match) {
            this.createdPolicyId = match[1];
            return null;
          }
          // a cross-origin deployment does not expose the Location header to
          // the browser; the policy is found by its name instead
          return this.axios.get(url).then((list) => {
            const policy = (list.data.policies || []).find(
              (candidate) => candidate.name === name,
            );
            this.createdPolicyId = policy ? String(policy.id) : null;
          });
        })
        .catch((error) => {
          this.$toastr.w(
            this.failureText(
              error,
              error.response && error.response.status === 409
                ? 'A policy with this name already exists'
                : null,
            ),
          );
        });
    },
    addComment: function () {
      if (!this.comment || !this.canEdit) {
        return;
      }
      const post = () => {
        const url = `${this.$api.BASE_URL}/api/v2/component-analyses/${this.analysisId}/comments`;
        return this.axios.post(url, { comment: this.comment }).then(() => {
          this.comment = null;
          this.loadComments();
        });
      };
      if (this.analysisId) {
        return post().catch((error) => {
          this.$toastr.w(this.failureText(error));
        });
      }
      // a comment hangs on an analysis; a component without one gets its
      // record created here, holding no override and no details, so a
      // finding can be noted before any license decision is taken
      const analysesUrl = `${this.$api.BASE_URL}/api/v2/component-analyses`;
      return this.axios
        .put(analysesUrl, {
          project_uuid: this.projectUuid,
          purl: this.component.purl || null,
          group: this.component.group || null,
          name: this.component.name,
          version: this.component.version || null,
          license: null,
          license_expression: null,
          license_name: null,
          details: null,
        })
        .then(() =>
          this.axios.get(`${analysesUrl}?project=${this.projectUuid}`),
        )
        .then((response) => {
          const analysis = (response.data.analyses || []).find(
            this.identityMatches,
          );
          this.analysisId = analysis ? analysis.id : null;
          return this.analysisId ? post() : null;
        })
        .catch((error) => {
          this.$toastr.w(this.failureText(error));
        });
    },
  },
};
</script>
