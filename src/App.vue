<template>
  <div>
    <header class="app-header">
      <img src="./assets/logo.png" alt="Logo" class="logo" />
      <h1>Team Raketa Jira AI Test Case Generator</h1>
    </header>
    <section class="filters card">
      <div class="filter-group">
        <label>
          Assigned To:
          <input v-model="assignedTo" type="text" placeholder="Enter assignee" />
        </label>
        <label>
          Area Path:
          <input v-model="areaPath" type="text" placeholder="Enter area path" />
        </label>
  <button class="download" @click="downloadTestCases">Download</button>
      </div>
      <button class="secondary" @click="askAiModel">Ask AI to identify its model</button>
      <div v-if="aiModelResponse" class="ai-model-response">
        <b>AI Model Response:</b> {{ aiModelResponse }}
      </div>
    </section>
    <section class="main-content">
      <div v-if="loading" class="loading-spinner">
        <span class="spinner"></span>
        <span class="loading-text">Loading...</span>
      </div>
      <div v-else-if="error" class="alert">Error: {{ error }}</div>
      <ul v-else class="issue-list card">
        <li v-for="issue in issues" :key="issue.id">
          <button class="issue-link" @click="fetchIssueDetails(issue.key)">
            <strong>{{ issue.key }}</strong>: {{ issue.fields.summary }}
          </button>
        </li>
      </ul>
      <div v-if="loadingDetails" class="loading-spinner">
        <span class="spinner"></span>
        <span class="loading-text">Generating test cases...</span>
      </div>
      <div v-else-if="selectedIssue" class="card issue-details">
        <h3>Issue Details: {{ selectedIssue.key }}</h3>
        <p><b>Summary:</b> {{ summary }}</p>
        <p><b>Description:</b> {{ descriptionContent }}</p>
        <div v-if="aiTestCaseResponse">
          <h4>AI-Generated Test Cases (via /ai/chat)</h4>
          <pre class="ai-response">{{ aiTestCaseResponse }}</pre>
          <div v-if="parsedTestCases.length">
            <h4>Test Cases (User Friendly)</h4>
            <div v-for="(tc, idx) in parsedTestCases" :key="idx" class="testcase-card">
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
        <button class="secondary" @click="selectedIssue = null">Close</button>
      </div>
    </section>
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
      loadingDetails: false,
      error: null,
      selectedIssue: null,
      summary: '',
      descriptionContent: '',
      testCases: '',
      aiModelResponse: '',
      aiTestCasePrompt: `Please output content as json`,
      aiTestCaseResponse: '',
      parsedTestCases: [],
      assignedTo: 'valentina.horvatic <valentina.horvatic@keepersolutions.com>',
      areaPath: 'accountsiq\\Group Squad'
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
      this.loadingDetails = true;
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
      } finally {
        this.loadingDetails = false;
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
      if (!this.parsedTestCases.length) return;
      const csvRows = [];
      // Header
      csvRows.push('Work Item Type,Title,Test Step,Step Action,Step Expected,Area Path,Assigned To,State');
      // For each test case
      this.parsedTestCases.forEach(tc => {
        // First row: Test Case info
        csvRows.push([
          '"Test Case"',
          `"${tc.title || ''}"`,
          '""', '""', '""',
          `"${this.areaPath}"`,
          `"${this.assignedTo}"`,
          '"Design"'
        ].join(','));
        // Steps
        if (tc.steps && tc.steps.length) {
          tc.steps.forEach((step, idx) => {
            csvRows.push([
              '""', '""',
              `"${step.stepNumber || idx + 1}"`,
              `"${step.action || ''}"`,
              `"${step.expectedResult || ''}"`,
              '""', '""', '""'
            ].join(','));
          });
        }
      });
      const csvContent = csvRows.join('\r\n');
      const blob = new Blob([csvContent], { type: 'text/csv' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = 'test-cases.csv';
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      URL.revokeObjectURL(url);
    }
  }
}
</script>

<style>
:root {
  --primary: #35495e;
  --accent: #42b883;
  --secondary: #e3eaf3;
  --danger: #e74c3c;
  --text: #35495e;
  --bg: #f6fafd;
  --card: #fff;
  --border: #b7c7d9;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

body {
  background: var(--bg);
  font-family: 'Inter', Avenir, Helvetica, Arial, sans-serif;
  color: var(--text);
}

#app {
  max-width: 900px;
  margin: 2.5rem auto;
  background: var(--card);
  border-radius: 12px;
  box-shadow: 0 2px 16px rgba(53,73,94,0.08);
  padding: 2rem 2.5rem;
  text-align: center;
}

.app-header {
  display: flex;
  align-items: center;
  gap: 1.2rem;
  margin-bottom: 2rem;
  justify-content: center;
}

.logo {
  width: 3rem;
  height: 3rem;
  border-radius: 8px;
}

.filters {
  margin-bottom: 2rem;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 1rem;
}

.filter-group {
  display: flex;
  gap: 3rem;
  align-items: center;
}

.main-content {
  margin-top: 1rem;
}

h1, h2, h3, h4 {
  color: var(--primary);
  margin-bottom: 0.5rem;
}

button {
  background: linear-gradient(90deg, var(--primary) 60%, var(--accent) 100%);
  color: var(--card);
  border: none;
  border-radius: 6px;
  padding: 0.5rem 1.2rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s, color 0.2s;
  font-size: 1rem;
  box-shadow: 0 1px 4px rgba(53,73,94,0.07);
}

button.download {
  background: var(--primary);
  color: var(--card);
}

button.download:hover, button.download:focus {
  background: var(--accent);
}

button.secondary {
  background: var(--card);
  color: var(--primary);
  border: 1px solid var(--primary);
}

button.secondary:hover {
  background: var(--primary);
  color: var(--secondary);
}

input[type="text"] {
  border: 1px solid var(--border);
  border-radius: 6px;
  padding: 0.4rem 0.8rem;
  font-size: 1rem;
  transition: border 0.2s;
  margin-left: 0.5rem;
  background: var(--card);
  color: var(--primary);
}

input[type="text"]:focus {
  border-color: var(--primary);
  outline: none;
  background: var(--bg);
}

ul.issue-list {
  list-style: none;
  padding: 1rem;
  margin: 0;
  text-align: left;
}

li {
  margin-bottom: 0.5rem;
}

.issue-link {
  background: none;
  border: none;
  color: var(--primary);
  cursor: pointer;
  text-align: left;
  font-size: 1.05rem;
  padding: 0.5rem;
  transition: color 0.2s, background 0.2s;
  border-radius: 4px;
  box-shadow: none;
}

.issue-link:hover {
  color: #fff;
  background: var(--primary);
  text-decoration: none;
}

.card {
  background: var(--secondary);
  border-radius: 8px;
  box-shadow: 0 1px 4px rgba(53,73,94,0.07);
  padding: 1.2rem 1.5rem;
  margin-bottom: 1.5rem;
  text-align: left;
}

.issue-details {
  margin: 2rem auto 0 auto;
}

.testcase-card {
  background: #e3eaf3;
  border-radius: 6px;
  padding: 1rem;
  margin-bottom: 1rem;
  border: 1px solid var(--border);
}

.ai-response {
  background: #f4f4f4;
  padding: 1rem;
  border-radius: 4px;
  white-space: pre-wrap;
  margin-bottom: 1rem;
}

.ai-model-response {
  margin-top: 1rem;
  color: var(--primary);
}

.loading-spinner {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: var(--primary);
  font-weight: 600;
  margin: 2rem 0 0;
  min-height: 80px;
}

.spinner {
  width: 2.5rem;
  height: 2.5rem;
  border: 4px solid var(--secondary);
  border-top: 4px solid var(--primary);
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-bottom: 1rem;
}

.loading-text {
  color: var(--primary);
  font-size: 1.1rem;
  font-weight: 500;
}

.alert {
  background: var(--danger);
  color: #fff;
  padding: 0.7rem 1rem;
  border-radius: 6px;
  margin-bottom: 1rem;
  text-align: left;
}
</style>
