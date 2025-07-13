---
title: 'Setting Timestamp in Second-Level Precision in Google Photo'
date: 2025-07-11
permalink: /posts/2025/07/google-photo/
tags:
  - tool
  - tutorial
---

Users of [Google Photo](https://www.google.com/photos/about/) should be aware that although the photo information interface only displays a time in minutes, photos taken within one minute are *usually* sorted correctly in chronological order. While some people might think they are sorted by filename or upload order, the truth is that they are still sorted by time but using an internal millisecond timestamp.

This internal timestamp can be problematic when users manually specify the date & time of a photo using Google Photo interface because this will truncate the timestamp to minute[^1], resulting in a problem known as wrong order of photos with “the same” timestamp[^2][^3]. Despite reported by many users, to the best of my knowledge, no solution or workaround has been proposed by users, nor has Google revealed any plan to fix it. This article studies the mechanism of the internal timestamp and then proposes a method to set timestamps with precision up to the second in Google Photo.

# Mechanism of the Internal Timestamp

Though the internal timestamp is crucial in photo sorting, the mechanism of it is very subtle and inconsistent. Because the algorithm used by Google Photo is proprietary, this section only describes a possible mechanism based on my experiments.

## Creation

The internal timestamp is created when a photo is uploaded, based on the EXIF data of the picture file, or the modification time recorded in the filesystem if the EXIF data is missing. There are many EXIF fields related to time, such as `CreateDate`, `SubSecCreateDate` and `SubSecTime`, and each of them could have `Original` and `Digitized` variants. It is not clean which field(s) does Google Photo look for, but eventually, it will obtain a millisecond `timestamp` and a `timezone`. For instance, if the EXIF has field `SubSecDateTimeOriginal` equaling to `2025:07:10 20:29:22.381+00:00`, then the `timestamp` will be `2025-07-10T20:29:22.381` and the `timezone` will be `+00:00`.

Because all photos are not perfectly timestamped, Google Photo has to handle missing data. The `timestamp` part is easy: the millisecond part defaults to `000` and it is thus always possible to obtain a timestamp. However, the `timezone` part is far more complex: when relevant EXIF fields are missing, Google Photo will infer it from GPS data and if GPS data is also missing, Google Photo will use the timezone of the uploader's IP address. Now we have three sources of timezone: (A) timezone recorded in `SubSecCreateDate` or `SubSecDateTimeOriginal` in the EXIF data; (B) the GPS coordinate in the EXIF data; (C) the IP address of the uploader. It is actually very common for these three sources to be inconsistent. For example, using UTC worldwide may result in conflict between sources (A) and (B); uploading photos after returning from an international trip may result in conflict between sources (A) and (C). In the presence of a conflict, source (B) has the highest priority, followed by source (A), and source (C) is least significant.

The story doesn't end after Google Photo has determined the `timestamp` and the `timezone` fields. Recall that when timezone sources (A) and (B) are inconsistent, Google Photo will use source (B) as the eventual timezone source, but will it adjust `timestamp` based on the new timezone? The answer is (surprisingly) *maybe*, i.e., sometimes will and sometimes won't. Again, using the example of the photo taken on `2025:07:10 20:29:22.381+00:00`, now suppose this photo's EXIF indicates that it was taken in New York whose timezone in `-04:00`. Because source (B) has higher priority, the eventual `timezone` will be `-04:00`, but the `timestamp` may be either (1) converted to `2025-07-10T16:29:22.381` (which is expected), or (2) persevered as `2025-07-10T20:29:22.381` (which is incorrect). In fact, both behavior (1) and behavior (2) happen frequently to me that I have to manually edit the “date & time” of approximately one third of my photos.

Once created, `timezone` and `timezone` will be stored separately as internal meta data *independent* of the picture file.

## Sorting Photos

Besides being truncated to minute and then displayed to the user, the *only* function of the internal timestamp is to sort photos in chronological order. Unfortunately, Google did not even implement this only function correctly. When there are multiple photos in different timezone, the expected behavior is sorting them in the absolute time rather than local time, e.g., `2025-07-10T20:00:00+00:00` is prior to (happens earlier) `2025-07-10T17:00:00-04:00` despite a greater value in local time. The web version of Google Photo behaves as expected, but its mobile app mistakenly sorts photos in local time. Sorting in local time means photos taken on a cross Pacific flight or in a town near timezone border will be missed up.

## Editing

The non-deterministic behavior due to the conflict between sources (A) and (B) is my major motivation to manually edit the timestamps of my photos. For other people, the most common motivation is that source (C) is often incorrect[^4][^5].

When editing the date & time of a photo, the interface only shows the time in minutes, and after editing, the second and millisecond part of the timestamp will be *reset to zero*. As a result, if there are two photos taken within one minutes, one with the automatically created internal timestamp and the other with manually edited timestamp, the second one will be prior to the first one regarding of their true taken time (unless in an extremely rare case where the first photo was taken exactly on the zeroth millisecond of that minute).

Google Photo also allows editing in batch. When editing the date & time of multiple photos at the same time, the user could choose to specify the timestamp of the first photo (the photo taken the earliest) and preserve the relative time between photos. In this case, the timestamp of the first photo will be truncated to minute, but the timestamp difference between the first photo and the remaining photos will be preserved at second-level (I initially thought it would be millisecond-level, but further experiments show that it is only second-level, and the timestamp of the remaining photos will be truncated to second). As a result, if the first photo has timestamp `13:01:32`, the timestamp of all photos will be subtracted by 32s. Consequently, a photo take at `17:01:10` will now have timestamp `17:00:38`. This subtraction is sufficient to cause the timestamp of a photo has a wrong minute field.

Editing the internal timestamp will not change the picture file.

# Precisely Editing Timestamps

Although the behavior of truncating during editing is annoying, it is actually possible to make use of the feature to precisely set the timestamp of a photo.

## Correcting Timezone

Before discussing how to specify the timestamp of a photo, let's first study a more common scenario—correcting the wrong timezone inferred during uploading[^1]. Although we could easily change the timestamp and timezone of photos in batch, doing this naively will result in the subtraction of timestamp.

Observing that the subtraction phenomenon changes the second field of the first photo to zero, if the field is already zero, then this phenomenon will not happen. We don't have to scarify any photos to make the first photo have zero second field; instead, we can upload a dummy picture, edit its timestamp to be prior to the first photo, and use the dummy photo to change the timestamps of useful photos in batch. Summarizing in pseudocode (because there is no Google Photo API to manage existing photos, it is very tricky to write a machine-executable script), the procedure is:

```
let S = the set of photos to edit in batch
create dummy.jpg (by using painting software or taking a photo)
upload dummy.jpg
let t = min(S.timestamp)
edit timestamp of dummy.jpg to t - 1min (this truncates dummy's timestamp and ensures dummy is the first photo)
select (S ∪ dummy.jpg)
edit timestamp (including timezone) of selected photos to the desired value
delete dummy.jpg
```

## Setting Timestamp

We can also precisely specify the timestamp (up to the second) using the dummy picture technique, but this time, we need to edit the EXIF data of the dummy picture (a good cross-platform tool is [ExifTool](https://exiftool.org/)).

The process consists of two phases: setting the timestamp to a value greater than the target value, and subtracting the timestamp to the exact value. Suppose we want to set the timestamp of photo.jpg to `2025-07-10T20:29:22`. We first set it to 2025-07-10 20:30, the approximation of the target timestamp by rounding towards infinity. Now, we need to subtract the timestamp by 38s. This requires uploading a dummy picture with `SubSecDateTimeOriginal`=`2025-07-10T20:29:38.000+00:00` (timezone is added to prevent using source (C)). Using the `convert` command from [ImageMagick](http://www.imagemagick.org/) and the ExifTool, this picture can be created by:

```sh
convert -size 128x128 xc:red dummy.jpg
exiftool -SubSecDateTimeOriginal="2025-07-10T20:29:38.000+00:00" dummy.jpg
```

The final step is selecting dummy.jpg and photo.jpg, and change the data & time of selected photos to 2025-07-10 20:29.

Due to the behavior of batch editing, this method can only set the timestamp to the zeroth millisecond of a second, i.e., the actual timestamp of photo.jpg will become `2025-07-10T20:29:22.000`. To verify this claim, we can upload two photos with timestamps `2025-07-10T20:29:21.999` and `2025-07-10T20:29:21.001`, respectively, and photo.jpg should be sorted between the two photos.

### Generating dummy.jpg

Since manually create dummy.jpg described above is tedious, I wrote a script to automate this process.

```bash
#!/bin/bash

# Usage: ./dummy.sh <date> <time>
# Example: ./dummy.sh 2024-07-10 20:29:22

DATE=$1
FULL_TIME=$2
TIME=${FULL_TIME:0:5}
SECOND=${FULL_TIME:6:2}
if [[ ${SECOND:0:1} -eq '0' ]]; then
    OFFSET=$(( 60 - ${SECOND:1:1} ))
else
    OFFSET=$(( 60 - $SECOND ))
fi

FILE=${DATE}T$FULL_TIME.jpg
TIMESTAMP="$(echo $DATE | sed s/-/:/g) $TIME:$OFFSET.000+00:00"

convert -background white -fill black -pointsize 128 label:"$DATE\n$FULL_TIME" $FILE

exiftool -SubSecDateTimeOriginal="$TIMESTAMP" $FILE -overwrite_original
```

To use the script, the user specifies the target timestamp of the useful picture, and the script will calculate the timestamp of the dummy picture and generate a picture containing the text of the target timestamp.

# Conclusion and Limitation

I have proposed a method to set the timestamp of a photo in Google Photo to second-level precision, but this method will truncate the millisecond part of the timestamp to 0. This method is good enough for most daily use but can still miss up photos taken within one second.

# References

[^1]: Yang-z. The seconds value of timestamp get lost after adjusting the time of photos or videos on the web. 2018. https://support.google.com/photos/thread/660932/the-seconds-value-of-timestamp-get-lost-after-adjusting-the-time-of-photos-or-videos-on-the-web?hl=en Accessed: 2025-07-13
[^2]: Adam A. Change the order of photos with the same timestamp. 2019. https://support.google.com/photos/thread/1032721/change-the-order-of-photos-with-the-same-timestamp?hl=en Accessed: 2025-07-10
[^3]: Martijn Groothuis. Google photos wrong order for sequence of photos. 2019. https://support.google.com/photos/thread/10211279?hl=en Accessed: 2025-07-10
[^4]: nontavit. Google Photos sorting and time zone issue. 2017. https://medium.com/@nontavit/google-photos-and-time-zone-issue-b2e2d20645b0 Accessed: 2025-07-10
[^5]: -SpaghettiCat-. Google Photos Not Sorting In Correct Chronological Order. 2021. https://www.reddit.com/r/google/comments/r3mets/google_photos_not_sorting_in_correct/ Accessed: 2025-07-10

