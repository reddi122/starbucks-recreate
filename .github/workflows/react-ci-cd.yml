name: React App CI/CD

on:
  workflow_dispatch:

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    env:
      SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      NEXUS_URL: ${{ secrets.NEXUS_URL }}
      NEXUS_USERNAME: ${{ secrets.NEXUS_USERNAME }}
      NEXUS_PASSWORD: ${{ secrets.NEXUS_PASSWORD }}
      NEXUS_REPO: starbugs-app
      NEXUS_GROUP: com/web/starbugs
      NEXUS_ARTIFACT: starbugs-app
      NGINX_HOST: ${{ secrets.NGINX_HOST }}
      NGINX_PATH: ${{ secrets.NGINX_PATH }}
      NGINX_SSH_KEY: ${{ secrets.NGINX_SSH_KEY }}

    steps:
      # === Stage 1: Checkout Code ===
      - name: 📦 Checkout source
        uses: actions/checkout@v4

      # === Stage 2: Install Dependencies ===
      - name: 📥 Install npm dependencies
        run: |
          npm install
          echo "✅ Dependencies installed!"

      # === Stage 3: SonarQube Analysis ===
      - name: 🔍 SonarQube Analysis
        run: |
          npm install -g sonarqube-scanner || npm install --save-dev sonarqube-scanner
          npx sonar-scanner \
            -Dsonar.projectKey=starbugs-app-js \
            -Dsonar.projectName="starbugs App JS" \
            -Dsonar.projectVersion=0.0.${{ github.run_number }} \
            -Dsonar.sources=src \
            -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
            -Dsonar.sourceEncoding=UTF-8 \
            -Dsonar.host.url=${{ env.SONAR_HOST_URL }} \
            -Dsonar.login=${{ env.SONAR_TOKEN }}

      # === Stage 4: Build React App ===
      - name: 🏗 Build React App
        run: |
          npm run build
          echo "✅ Build Completed!"
          ls -lh build || true

      # === Stage 5: Package Artifact ===
      - name: 📦 Package Build
        run: |
          VERSION="0.0.${{ github.run_number }}"
          tar -czf ${NEXUS_ARTIFACT}-${VERSION}.tar.gz -C build .
          echo "✅ Package created: ${NEXUS_ARTIFACT}-${VERSION}.tar.gz"
          ls -lh *.tar.gz

      # === Stage 6: Upload to Nexus ===
      - name: 📤 Upload Artifact to Nexus
        run: |
          set -e
          VERSION="0.0.${{ github.run_number }}"
          TARBALL="${NEXUS_ARTIFACT}-${VERSION}.tar.gz"
          echo "Uploading $TARBALL to Nexus..."
          curl -v -u "${{ env.NEXUS_USERNAME }}:${{ env.NEXUS_PASSWORD }}" \
            --upload-file "$TARBALL" \
            "${{ env.NEXUS_URL }}/repository/${{ env.NEXUS_REPO }}/${{ env.NEXUS_GROUP }}/${{ NEXUS_ARTIFACT }}/${VERSION}/${TARBALL}"
          echo "✅ Artifact uploaded successfully to Nexus!"

      # === Stage 7: Deploy to Nginx ===
      - name: 🚀 Deploy to Nginx via SSH
        uses: appleboy/ssh-action@v0.1.6
        with:
          host: ${{ env.NGINX_HOST }}
          key: ${{ env.NGINX_SSH_KEY }}
          script: |
            VERSION="0.0.${{ github.run_number }}"
            TARBALL="${NEXUS_ARTIFACT}-${VERSION}.tar.gz"
            TMP_DIR="/tmp/deploy_build"

            # Clean old files
            rm -rf $TMP_DIR
            mkdir -p $TMP_DIR

            # Download artifact from Nexus
            curl -f -u "${{ env.NEXUS_USERNAME }}:${{ env.NEXUS_PASSWORD }}" \
              -O "${{ env.NEXUS_URL }}/repository/${{ env.NEXUS_REPO }}/${{ env.NEXUS_GROUP }}/${{ NEXUS_ARTIFACT }}/${VERSION}/${TARBALL}"

            # Extract to temporary dir
            tar -xzf "$TARBALL" -C $TMP_DIR

            # Copy files to Nginx web root
            sudo rm -rf ${{ env.NGINX_PATH }}/*
            sudo mkdir -p ${{ env.NGINX_PATH }}
            sudo cp -r $TMP_DIR/* ${{ env.NGINX_PATH }}/
            sudo chown -R www-data:www-data ${{ env.NGINX_PATH }}

            echo "✅ Deployment successful! Application live on Nginx!"
