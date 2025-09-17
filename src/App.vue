<template>
  <div>
    <h2>Jira Issues</h2>
    <div style="margin-bottom:1em;">
      <label>
        Assigned To:
        <input v-model="assignedTo" type="text" placeholder="Enter assignee" style="margin-right:1em;" />
      </label>
      <label>
        Area Path:
        <input v-model="areaPath" type="text" placeholder="Enter area path" />
      </label>
      <button @click="downloadTestCases" style="margin-left:1em;">Download</button>
    </div>
    <button @click="askAiModel" style="margin-bottom:1em;">Ask AI to identify its model</button>
    <div v-if="aiModelResponse" style="margin-bottom:1em;">
      <b>AI Model Response:</b> {{ aiModelResponse }}
    </div>
    <div v-if="loading">Loading issues...</div>
    <div v-else-if="error">Error: {{ error }}</div>
    <ul v-else>
      <li v-for="issue in issues" :key="issue.id">
        <button @click="fetchIssueDetails(issue.key)" style="background:none;border:none;padding:0;color:#3498db;cursor:pointer;text-align:left;">
          <strong>{{ issue.key }}</strong>: {{ issue.fields.summary }}
        </button>
      </li>
    </ul>
    <div v-if="selectedIssue" style="margin-top:2em;text-align:left;max-width:600px;margin-left:auto;margin-right:auto;">
      <h3>Issue Details: {{ selectedIssue.key }}</h3>
      <p><b>Summary:</b> {{ summary }}</p>
      <p><b>Description:</b> {{ descriptionContent }}</p>
      <div v-if="aiTestCaseResponse">
        <h4>AI-Generated Test Cases (via /ai/chat)</h4>
        <pre style="background:#f4f4f4;padding:1em;border-radius:4px;white-space:pre-wrap;">{{ aiTestCaseResponse }}</pre>
        <div v-if="parsedTestCases.length">
          <h4>Test Cases (User Friendly)</h4>
          <div v-for="(tc, idx) in parsedTestCases" :key="idx" style="margin-bottom:1em;padding:1em;background:#eaf6fb;border-radius:4px;">
            <b>{{ tc.title }}</b>
            <div><b>Description:</b> {{ tc.description }}</div>
            <div><b>Preconditions:</b> {{ tc.preconditions }}</div>
            <div v-if="tc.steps && tc.steps.length">
              <b>Steps:</b>
              <ol>
                <li v-for="step in tc.steps" :key="step.stepNumber">
                  <span><b>Action:</b> {{ step.action }}</span><br/>
                  <span><b>Expected Result:</b> {{ step.expectedResult }}</span>
                </li>
              </ol>
            </div>
          </div>
        </div>
      </div>
      <button @click="selectedIssue = null">Close</button>
    </div>
  </div>
</template>

<script>
import axios from 'axios'
/* eslint-disable */
export default {
  name: 'App',
  data() {
    return {
      issues: [],
      loading: false,
      error: null,
      selectedIssue: null,
      summary: '',
      descriptionContent: '',
      testCases: '',
      aiModelResponse: '',
      aiTestCasePrompt: `Please output content as json`,
      aiTestCaseResponse: '',
      parsedTestCases: [],
      assignedTo: '',
      areaPath: ''
    }
  },
  mounted() {
    this.fetchJiraIssues();
  },
  methods: {
    async fetchJiraIssues() {
      this.loading = true;
      this.error = null;
      try {
        const res = await axios.get('http://localhost:4000/jira/issues');
        this.issues = res.data;
        console.log('Fetched issues:', this.issues);
      } catch (err) {
        this.error = err.message || 'Failed to fetch issues';
      } finally {
        this.loading = false;
      }
    },
    async fetchIssueDetails(issueKey) {
      this.selectedIssue = null;
      this.summary = '';
      this.descriptionContent = '';
      this.testCases = '';
      this.aiTestCaseResponse = '';
      this.parsedTestCases = [];
      try {
        const res = await axios.get(`http://localhost:4000/jira/issue/${issueKey}`);
        const issue = res.data;
        this.selectedIssue = issue;
        this.summary = issue.fields.summary || '';
        // Handle both plain text and Atlassian Document Format for description
        const desc = issue.fields.description;
        if (typeof desc === 'string') {
          this.descriptionContent = desc;
        } else if (desc && desc.content && Array.isArray(desc.content)) {
          this.descriptionContent = desc.content.map(block => {
            if (block.content && Array.isArray(block.content)) {
              return block.content.map(inline => inline.text || '').join('');
            }
            return '';
          }).join('\n');
        } else {
          this.descriptionContent = 'No description.';
        }
        // Call AI chat endpoint with description + extra instructions
        const aiPrompt = `${this.descriptionContent}\n\n${this.aiTestCasePrompt}`;
        const aiRes = await axios.post('http://localhost:4000/ai/chat', {
          prompt: aiPrompt,
          model: 'hackathon-team-raketa' // Specify the model here
        });
        this.aiTestCaseResponse = aiRes.data.response;
        // Try to extract and parse JSON from AI response
        this.parsedTestCases = [];
        try {
          // Find first [ or { and last ] or }
          const text = this.aiTestCaseResponse;
          let jsonStart = text.indexOf('[');
          let jsonEnd = text.lastIndexOf(']');
          if (jsonStart === -1 || jsonEnd === -1) {
            jsonStart = text.indexOf('{');
            jsonEnd = text.lastIndexOf('}');
          }
          if (jsonStart !== -1 && jsonEnd !== -1 && jsonEnd > jsonStart) {
            const jsonString = text.slice(jsonStart, jsonEnd + 1);
            const parsed = JSON.parse(jsonString);
            this.parsedTestCases = Array.isArray(parsed) ? parsed : [parsed];
          }
        } catch (e) {
          this.parsedTestCases = [];
        }
      } catch (err) {
        this.error = err.message || 'Failed to fetch issue details';
      }
    },
    async askAiModel() {
      this.aiModelResponse = '';
      try {
        const res = await axios.post('http://localhost:4000/ai/chat', {
          prompt: 'Identify your model. Just reply with the model name.',
          model: 'hackathon-team-raketa' // Specify the model here
        });
        this.aiModelResponse = res.data.response;
      } catch (err) {
        this.aiModelResponse = 'Failed to get AI model.';
      }
    },
    downloadTestCases() {
      // Empty method for download action
    }
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
