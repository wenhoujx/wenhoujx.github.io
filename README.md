# My personal website

## Gomoku
most basic rules with 10 rollbacks.


## Deploy

Upload to gcloud bucket: app engine > Settings > Application Settings > Default Cloud Storage Bucket

DELETE existing build/ folder from glcoud
Upload `npm run build` > the folder build/ and app.yaml 

Open gcloud terminal (top right)
may need to create wenhoujx-website folder 
```
# may need authorize
mkdir ./wenhoujx-website
gsutil rsync -r gs://bucket-name ./wenhoujx-website
# gsutil rsync -r gs://cellular-virtue-287501.appspot.com ./wenhoujx-website
cd ./wenhoujx-website
gcloud app deploy
# make sure billing account is connected and enabled.
```

## ideas: 

- turn chinese charater to icon with different px size.
- time tracking app
- goodbye pets 

## dev
```sh 
# install package 
npm i react-bootstrap bootstrap 

# start server 
npm run start
``` 
