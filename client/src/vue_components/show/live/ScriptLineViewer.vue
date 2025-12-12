<template>
  <b-container
    v-once
    ref="lineContainer"
    class="mx-0"
    style="margin: 0; padding: 0"
    fluid
  >
    <b-row
      v-if="needsIntervalBanner"
      class="interval-header"
    >
      <b-col
        cols="12"
        class="interval-banner"
      >
        <b-alert
          show
          variant="warning"
          style="margin: 0"
        >
          <h3> {{ previousActLabel }} - Interval</h3>
        </b-alert>
      </b-col>
      <b-col
        v-if="isScriptLeader"
        cols="12"
        class="d-flex align-items-center justify-content-center"
        style="padding-top: 0.5rem; padding-bottom: 0.5rem;"
      >
        <b-button
          variant="primary"
          @click.stop="startInterval"
        >
          Start Interval
        </b-button>
      </b-col>
    </b-row>
    <b-row
      v-if="needsActSceneLabel"
      class="act-scene"
    >
      <b-col
        cols="2"
        class="cue-column text-right font-weight-bold cue"
      >
        <span>{{ actLabel }}</span>
      </b-col>
      <b-col class="line-part text-left font-weight-bold cue">
        <span>{{ sceneLabel }}</span>
      </b-col>
    </b-row>
    <template v-for="cue in cues">
      <b-row :key="`cue_${cue.id}`">
        <b-col
          cols="2"
          class="cue-column line-part text-right font-weight-bold cue"
          :style="{color: cueBackgroundColour(cue)}"
        >
          <span>
            {{ cuePrefix(cue) }}
          </span>
        </b-col>
        <b-col
          cols="10"
          class="line-part text-left font-weight-bold cue"
          :style="{color: cueBackgroundColour(cue)}"
        >
          <span>
            {{ cue.ident }}
          </span>
        </b-col>
      </b-row>
    </template>
    <b-row v-if="cueAddMode && !isWholeLineCut(line)">
      <b-col
        cols="12"
        class="line-part text-center cue-add-row"
      >
        <b-button
          variant="success"
          size="sm"
          @click.stop="addNewCue"
        >
          <b-icon-plus-square-fill /> Add Cue
        </b-button>
      </b-col>
    </b-row>
    <b-row>
      <template v-if="line.stage_direction">
        <b-col cols="2" class="cue-column" />
        <b-col
          :key="`line_${lineIndex}_stage_direction`"
          class="line-part text-left"
        >
          <i
            class="viewable-line"
            :style="stageDirectionStyling"
          >
            <template
              v-if="stageDirectionStyle != null && stageDirectionStyle.text_format === 'upper'"
            >
              {{ line.line_parts[0].line_text | uppercase }}
            </template>
            <template
              v-else-if="stageDirectionStyle != null && stageDirectionStyle.text_format === 'lower'"
            >
              {{ line.line_parts[0].line_text | lowercase }}
            </template>
            <template v-else>
              {{ line.line_parts[0].line_text }}
            </template>
          </i>
        </b-col>
      </template>
      <template v-else>
        <template
          v-for="(part, index) in line.line_parts"
        >
          <template v-if="!isCueLine(part) || hasVisibleText(part)">
            <b-col
              :key="`char_${lineIndex}_part_${index}`"
              cols="2"
              class="cue-column line-part text-right"
              :class="{
                'cut-line-part': cuts.indexOf(part.id) !== -1,
                'line-part-a': lineIndex%2==0,
                'line-part-b': lineIndex%2==1
              }"
            >
              <p v-if="needsHeadings[index]">
                <template v-if="part.character_id != null">
                  {{ characters.find((char) => (char.id === part.character_id)).name }}
                </template>
                <template v-else>
                  {{ characterGroups.find((char) => (char.id === part.character_group_id)).name }}
                </template>
              </p>
            </b-col>
            <b-col
              :key="`text_${lineIndex}_part_${index}`"
              cols="10"
              class="line-part text-left"
              :class="{
                'cut-line-part': cuts.indexOf(part.id) !== -1,
                'line-part-a': lineIndex%2==0,
                'line-part-b': lineIndex%2==1
              }"
            >
              <p class="viewable-line">
                {{ part.line_text }}
              </p>
            </b-col>
          </template>
        </template>
      </template>
    </b-row>
  </b-container>
</template>

<script>
import { mapGetters, mapActions } from 'vuex';
import { contrastColor } from 'contrast-color';

export default {
  name: 'ScriptLineViewer',
  events: ['last-line-change', 'first-line-change', 'start-interval'],
  props: {
    line: {
      required: true,
      type: Object,
    },
    lineIndex: {
      required: true,
      type: Number,
    },
    previousLine: {
      required: true,
      type: Object,
    },
    previousLineIndex: {
      required: true,
      type: Number,
    },
    acts: {
      required: true,
      type: Array,
    },
    scenes: {
      required: true,
      type: Array,
    },
    characters: {
      required: true,
      type: Array,
    },
    characterGroups: {
      required: true,
      type: Array,
    },
    cueTypes: {
      required: true,
      type: Array,
    },
    cues: {
      required: true,
      type: Array,
    },
    cuts: {
      required: true,
      type: Array,
    },
    stageDirectionStyles: {
      required: true,
      type: Array,
    },
    stageDirectionStyleOverrides: {
      required: true,
      type: Array,
    },
    isScriptLeader: {
      required: true,
      type: Boolean,
    },
    cueAddMode: {
      required: true,
      type: Boolean,
    },
  },
  data() {
    return {
      observer: null,
    };
  },
  computed: {
    needsHeadings() {
      let { previousLine } = this;
      let lineIndex = this.previousLineIndex;
      while (previousLine != null && (previousLine.stage_direction === true
          || this.isWholeLineCut(previousLine))) {
        [lineIndex, previousLine] = this.getPreviousLineForIndex(previousLine.page, lineIndex);
      }

      const ret = [];
      this.line.line_parts.forEach(function checkLinePartNeedsHeading(part) {
        if (previousLine == null
          || previousLine.line_parts.length !== this.line.line_parts.length) {
          ret.push(true);
        } else if (previousLine.act_id !== this.line.act_id || previousLine.scene_id !== this.line.scene_id) {
          ret.push(true);
        } else {
          const matchingIndex = previousLine.line_parts.find((prevPart) => (
            prevPart.part_index === part.part_index));
          if (matchingIndex == null) {
            ret.push(true);
          } else {
            ret.push(!(matchingIndex.character_id === part.character_id
              && matchingIndex.character_group_id === part.character_group_id));
          }
        }
      }, this);
      return ret;
    },
    needsHeadingsAny() {
      return this.needsHeadings.some((x) => (x === true));
    },
    needsHeadingsAll() {
      return this.needsHeadings.every((x) => (x === true));
    },
    needsActSceneLabel() {
      let { previousLine } = this;
      let lineIndex = this.previousLineIndex;
      while (previousLine != null && || this.isWholeLineCut(previousLine)) {
        [lineIndex, previousLine] = this.getPreviousLineForIndex(previousLine.page, lineIndex);
      }
      if (previousLine == null) {
        return true;
      }
      return !(previousLine.act_id === this.line.act_id
        && previousLine.scene_id === this.line.scene_id);
    },
    needsIntervalBanner() {
      let { previousLine, lineIndex } = this;
      while (previousLine != null && this.isWholeLineCut(previousLine)) {
        [lineIndex, previousLine] = this.getPreviousLineForIndex(previousLine.page, lineIndex);
      }
      if (previousLine == null) {
        return false;
      }
      return previousLine.act_id !== this.line.act_id;
    },
    previousActLabel() {
      return this.acts.find((act) => (act.id === this.previousLine.act_id)).name;
    },
    actLabel() {
      return this.acts.find((act) => (act.id === this.line.act_id)).name;
    },
    sceneLabel() {
      return this.scenes.find((scene) => (scene.id === this.line.scene_id)).name;
    },
    stageDirectionStyle() {
      const sdStyle = this.stageDirectionStyles.find(
        (style) => (style.id === this.line.stage_direction_style_id),
      );
      const override = this.stageDirectionStyleOverrides
        .find((elem) => elem.settings.id === sdStyle.id);
      if (this.line.stage_direction) {
        return override ? override.settings : sdStyle;
      }
      return null;
    },
    stageDirectionStyling() {
      if (this.line.stage_direction_style_id == null || this.stageDirectionStyle == null) {
        return {
          'background-color': 'darkslateblue',
          'font-style': 'italic',
        };
      }
      const style = {
        'font-weight': this.stageDirectionStyle.bold ? 'bold' : 'normal',
        'font-style': this.stageDirectionStyle.italic ? 'italic' : 'normal',
        'text-decoration-line': this.stageDirectionStyle.underline ? 'underline' : 'none',
        color: this.stageDirectionStyle.text_colour,
      };
      if (this.stageDirectionStyle.enable_background_colour) {
        style['background-color'] = this.stageDirectionStyle.background_colour;
      }
      return style;
    },
    ...mapGetters(['GET_SCRIPT_PAGE', 'SCRIPT_CUTS', 'USER_SETTINGS']),
  },
  mounted() {
    /* eslint-disable no-restricted-syntax */
    this.observer = new MutationObserver((mutations) => {
      for (const m of mutations) {
        const newValue = m.target.getAttribute(m.attributeName);
        this.$nextTick(() => {
          this.onClassChange(newValue, m.oldValue);
        });
      }
    });
    /* eslint-enable no-restricted-syntax */

    this.observer.observe(this.$refs.lineContainer, {
      attributes: true,
      attributeOldValue: true,
      attributeFilter: ['class'],
    });
  },
  destroyed() {
    this.observer.disconnect();
  },
  methods: {
    contrastColor,
    onClassChange(classAttrValue, oldClassAttrValue) {
      const classList = classAttrValue.split(' ');
      const oldClassList = oldClassAttrValue.split(' ');
      if (classList.includes('last-script-element') && !oldClassList.includes('last-script-element')) {
        this.$emit('last-line-change', this.line.page, this.lineIndex);
      }
      if (classList.includes('first-script-element') && !oldClassList.includes('first-script-element')) {
        let previousLine = null;
        if (this.previousLine != null) {
          previousLine = `page_${this.previousLine.page}_line_${this.previousLineIndex}`;
        }
        this.$emit('first-line-change', this.line.page, this.lineIndex, previousLine);
      }
    },
    cuePrefix(cue) {
      const cueType = this.cueTypes.find((cT) => (cT.id === cue.cue_type_id));
      return cueType.prefix;
    },
    cueLabel(cue) {
      const cueType = this.cueTypes.find((cT) => (cT.id === cue.cue_type_id));
      return `${cueType.prefix} ${cue.ident}`;
    },
    cueBackgroundColour(cue) {
      return this.cueTypes.find((cueType) => (cueType.id === cue.cue_type_id)).colour;
    },
    getPreviousLineForIndex(pageIndex, lineIndex) {
      if (lineIndex > 0) {
        return [lineIndex - 1, this.GET_SCRIPT_PAGE(pageIndex)[lineIndex - 1]];
      }
      let loopPageNo = pageIndex - 1;
      while (loopPageNo >= 1) {
        const loopPage = this.GET_SCRIPT_PAGE(loopPageNo);
        if (loopPage.length > 0) {
          return [loopPage.length - 1, loopPage[loopPage.length - 1]];
        }
        loopPageNo -= 1;
      }
      return [null, null];
    },
    isWholeLineCut(line) {
      return line.line_parts.every((linePart) => (this.SCRIPT_CUTS.includes(linePart.id)), this);
    },
    isCueLine(part) {
      const character = this.characters.find((char) => (char.id === part.character_id));
      return character && character.name === 'CUE';
    },
    hasVisibleText(part) {
      return part.line_text && part.line_text.replace(/\s/g, '').length > 0;
    },
    startInterval() {
      this.$emit('start-interval', this.acts.find((act) => (act.id === this.previousLine.act_id)).id);
    },
    addNewCue() {
      this.$emit('add-cue', this.line.id);
    },
  },
};
</script>

<style scoped>
  .cue-column {
    border-right: .1rem solid #3498db;
  }

  .cut-line-part {
    text-decoration: line-through;
  }

  .line-part {
    font-size: 1.5rem;
  }

  .cue {
    font-size: 2rem;
  }

  .line-part-a {
    color: white;
  }

  .line-part-b {
    color: gray;
  }

  .act-scene {
    color: #f401fe;
  }

  .current-line {
    background: #3498db54;
  }

  .interval-header {
    background: var(--body-background);
    margin-top: 1rem;
    padding-bottom: 1rem;
  }

  .interval-banner {
    margin-top: -1rem;
    margin-bottom: -1rem;
    padding-top: 1rem;
    padding-bottom: 1rem;
  }

  .cue-add-row {
    padding-top: 0.5rem;
    padding-bottom: 0.5rem;
  }
</style>
