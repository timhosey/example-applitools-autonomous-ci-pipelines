pipeline {
  agent any

  environment {
    AUTONOMOUS_BASE_URL = 'https://autonomous.applitools.com' // adjust if your tenant differs
    PLAN_ID             = 'YOUR_PLAN_ID'
  }

  stages {
    stage('Run Autonomous Test Plan') {
      steps {
        withCredentials([string(credentialsId: 'AUTONOMOUS_API_KEY', variable: 'APPLITOOLS_API_KEY')]) {
          // Shell script to trigger the autonomous plan run
          sh '''
            set -e

            echo "Triggering Autonomous plan run..."

            # Execute plan per Applitools docs: POST /api/plan/{plan_id}/execute?apiKey={personal_key}
            RESPONSE=$(curl -sS -X POST "$AUTONOMOUS_BASE_URL/api/plan/$PLAN_ID/execute?apiKey=$APPLITOOLS_API_KEY" \
              -H "Content-Type: application/json" \
              -d '{}')

            echo "Execute response: $RESPONSE"

            # Extract run id and read_token from result (response: { "result": { "id": "...", "read_token": "..." } })
            RUN_ID=$(echo "$RESPONSE" | jq -r '.result.id // .run_id // empty')
            READ_TOKEN=$(echo "$RESPONSE" | jq -r '.result.read_token // .read_token // empty')

            if [ -z "$RUN_ID" ] || [ "$RUN_ID" = "null" ]; then
              echo "Failed to retrieve run_id from execute response"
              exit 1
            fi

            echo "Run ID: $RUN_ID"

            # Persist values for later stages
            echo "RUN_ID=$RUN_ID"      >  autonomous_env.properties
            echo "READ_TOKEN=$READ_TOKEN" >> autonomous_env.properties
          '''
          script {
            def content = readFile file: 'autonomous_env.properties'
            content.split('\n').each { line ->
              def eq = line.indexOf('=')
              if (eq > 0) {
                def key = line.substring(0, eq).trim()
                def val = line.substring(eq + 1).trim()
                if (key == 'RUN_ID') env.RUN_ID = val
                if (key == 'READ_TOKEN') env.READ_TOKEN = val
              }
            }
          }
        }
      }
    }

    stage('Poll Autonomous Results') {
      steps {
        // Uses credential in Jenkins called AUTONOMOUS_API_KEY to get the API key
        withCredentials([string(credentialsId: 'AUTONOMOUS_API_KEY', variable: 'APPLITOOLS_API_KEY')]) {
          // Shell script to poll the results of the autonomous plan run
          sh '''
            if [ -z "$RUN_ID" ]; then
              echo "RUN_ID not set"
              exit 1
            fi

            echo "Polling results for run: $RUN_ID"

            MAX_ATTEMPTS=60      # up to ~30 minutes with 30s sleep
            SLEEP_SECONDS=30

            ATTEMPT=1
            STATUS="pending"

            while [ $ATTEMPT -le $MAX_ATTEMPTS ]; do
              echo "Attempt $ATTEMPT/$MAX_ATTEMPTS..."

              if [ -n "$READ_TOKEN" ]; then
                URL="$AUTONOMOUS_BASE_URL/api/result/$RUN_ID?apiKey=$APPLITOOLS_API_KEY&read_token=$READ_TOKEN"
              else
                URL="$AUTONOMOUS_BASE_URL/api/result/$RUN_ID?apiKey=$APPLITOOLS_API_KEY"
              fi

              RESULT_JSON=$(curl -sS -X GET "$URL" -w "\\n%{http_code}" 2>/dev/null) || true
              HTTP_CODE=$(echo "$RESULT_JSON" | tail -n1)
              RESULT_JSON=$(echo "$RESULT_JSON" | sed '$d')

              echo "Result payload:"
              echo "$RESULT_JSON"

              if [ -z "$RESULT_JSON" ] || [ "$HTTP_CODE" != "200" ]; then
                echo "Connection failure or non-200 (http_code=$HTTP_CODE), will retry..."
                STATUS="pending"
              else
                STATUS=$(echo "$RESULT_JSON" | jq -r '.result.status // .status // "pending"' 2>/dev/null) || STATUS="pending"
              fi

              echo "Current status: $STATUS"

              # Terminal states: passed, failed, aborted, unresolved
              if [ "$STATUS" = "passed" ] || [ "$STATUS" = "failed" ] || [ "$STATUS" = "aborted" ] || [ "$STATUS" = "unresolved" ]; then
                break
              fi

              echo "Still in progress, sleeping for $SLEEP_SECONDS seconds..."
              sleep $SLEEP_SECONDS
              ATTEMPT=$((ATTEMPT + 1))
            done

            if [ "$STATUS" = "passed" ]; then
              echo "Autonomous plan run passed."
              exit 0
            fi

            if [ $ATTEMPT -gt $MAX_ATTEMPTS ]; then
              echo "Timed out waiting for Autonomous plan to finish. Last status: $STATUS"
            elif [ "$STATUS" = "unresolved" ]; then
              echo "Autonomous plan finished with status: unresolved (visual differences or unresolved steps)."
            else
              echo "Autonomous plan finished with non-passing status: $STATUS"
            fi

            exit 1
          '''
        }
      }
    }
  }
}
