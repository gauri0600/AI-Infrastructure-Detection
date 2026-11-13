# Script walkthrough

A study guide for `code/infrastructure_detection.js`. Read it with the script open in the Earth Engine Code Editor so you can run each idea as you go.

## 1. The one-sentence version

Take a year of satellite images, squash them into one clean image, compute three "what is this surface?" indices per pixel, label pixels using simple thresholds, train a Random Forest on a sample of those labels, and let it label every pixel.

## 2. The pieces, in order

**Study area.** `ee.Geometry.Rectangle([74.5, 32.5, 75.0, 33.0])` is a box given as west, south, east, north in degrees.

**Image collection.** `ee.ImageCollection('COPERNICUS/S2_SR')` is every Sentinel-2 image ever taken. The three `.filter…` calls narrow it to this box, this year, and scenes under 20% cloud. `.select([...])` keeps five bands:

| Band | Colour | Why it is here |
|---|---|---|
| B2, B3, B4 | blue, green, red | what your eye sees |
| B8 | near-infrared (NIR) | healthy plants reflect it strongly |
| B11 | short-wave infrared (SWIR) | concrete and dry soil reflect it strongly |

**Median composite.** `.median()` takes, for each pixel, the middle value across all the images. Clouds are bright and shadows are dark, so both are outliers, and the median ignores them.

**Indices.** `normalizedDifference([A, B])` computes `(A − B) / (A + B)`, always between −1 and +1.

- NDVI = (NIR − Red) / (NIR + Red). Plants: high NIR, low red, so high NDVI.
- NDWI = (Green − NIR) / (Green + NIR). Water absorbs NIR, so high NDWI.
- NDBI = (SWIR − NIR) / (SWIR + NIR). Built surfaces: SWIR higher than NIR, so positive NDBI.

**Threshold labels.** Lines like `ndbi.gt(0.10).and(ndvi.lt(0.30))` produce a true/false image. The `.where(mask, n)` chain stamps class numbers 1 to 5 onto one image. Order matters: a later `.where` overwrites an earlier one where both masks are true.

**Sampling.** `stratifiedSample({numPoints: 100, classBand: 'landcover'})` picks up to 100 random pixels from each class value, including 0 (pixels that matched no rule).

**Random Forest.** 100 decision trees, each trained on a random half of the samples (`bagFraction: 0.5`), each split considering a random subset of features. The class with the most votes wins.

**Area.** `ee.Image.pixelArea()` gives each pixel's area in m². Masking it to one class and summing with `reduceRegion` gives that class's total area.

## 3. The most important thing to understand: the accuracy number

`classifier.confusionMatrix()` scores the model on its own training samples. That is called **resubstitution accuracy**, and it is close to 100% for almost any Random Forest. On top of that, the labels were created from NDVI, NDWI and NDBI, and the model is given those same three indices as inputs. It is being asked to rediscover the thresholds you wrote.

So the project demonstrates a complete, working classification pipeline. It does not yet demonstrate how accurate the map is. Knowing that distinction, and saying it before anyone asks, is what makes the project credible.

**Check yourself**
- Why does the median remove clouds but a mean would not?
- A pixel has NIR = 0.30 and Red = 0.10. What is its NDVI? (Answer: 0.20 / 0.40 = 0.5, vegetation.)
- In the console screenshot, the row for class 3 (water) is all zeros. What does that tell you about the sample? What could cause it: the `.where` order, the NDWI threshold, or the area simply having little open water at 10 m?
- Why is "accuracy on training data" not evidence the map is right?

## 4. Change it yourself

Do these in order, and record what the Console prints each time.

1. **Add a real train/test split.** After creating `trainingData`:

   ```js
   var withRandom = trainingData.randomColumn('random', 42);
   var trainSet = withRandom.filter(ee.Filter.lt('random', 0.7));
   var testSet  = withRandom.filter(ee.Filter.gte('random', 0.7));
   ```

   Train on `trainSet`, then:

   ```js
   var tested = testSet.classify(classifier);
   var testMatrix = tested.errorMatrix('landcover', 'classification');
   print('Test accuracy:', testMatrix.accuracy());
   ```

   This is still measured against threshold labels, but it is at least measured on pixels the model has not seen.

2. **Remove the three indices from `inputBands`** so the model only sees the five raw bands. Does test accuracy drop? By how much? That gap shows how much the model was leaning on the indices.

3. **Find out why water has no samples** and fix it.

4. **Collect 50 points per class by hand** using the Code Editor's geometry tools on the true-colour layer, and evaluate against those. That number is the first one you can honestly call accuracy.

When you have real numbers from steps 1 and 4, add them to the README with the date you ran them.

## 5. Explaining it in an interview

1. What problem does this solve, in two sentences?
2. Why Random Forest and not a neural network? (Small tabular feature set, few samples, fast to train, available in Earth Engine. This is classical machine learning, not deep learning.)
3. Where do the training labels come from, and what is the weakness of that?
4. What does your accuracy number measure, and what would you do to measure it properly?
