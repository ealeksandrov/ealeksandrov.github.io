---
layout: post
title: "Overlaying build information on top of app icon"
date: 2016-02-28 15:00:00 +0400
---

During development I prefer to keep separate dev/qa/production versions of my application installed on device. This can be simply achieved by using different `bundle id`s. But they all will look the same on device springboard. Earlier I stored icon name in [configs](http://aleksandrov.ws/2015/02/27/xcode-project-configuration/) to use different icons but it still require to manually create and add this icons. And it have no valuable information apart of differentiating between types of builds.

In this article I will show a script to overlay icon with information from application `Info.plist`, `xcconfig`s or `git` output.

<!-- more -->

## Available information

To extract information from application `Info.plist` use:

{% highlight shell %}
version=`/usr/libexec/PlistBuddy -c "Print CFBundleVersion" "${INFOPLIST_FILE}"`
{% endhighlight %}

For terminal output (like `git`):

{% highlight shell %}
branch=`${git} rev-parse --abbrev-ref HEAD`
{% endhighlight %}

For variables from `xcconfig`:

{% highlight shell %}
scheme_Name="${APP_SCHEME_NAME}"
{% endhighlight %}

## Tools

Tools required for image proccesing are: ImageMagick and ghostscript. You can use brew to quickly install them:

{% highlight shell %}
brew install imagemagick
brew install ghostscript
{% endhighlight %}

## Overlaying script

Add script to new build phase: `ProjectName > TargetName > Build Phases > New Run Script Phase`.

Script can be just dropped as a text here, but I prefer to store and it separately, so I am adding:

{% highlight shell %}
sh Scripts/iconVersioning.sh
{% endhighlight %}

![Adding Build Phase](/static/article-icon-overlaying/01.png)

To prevent overlaying icon on release builds it will exit on "Release" configuration:

{% highlight shell %}
if [ $CONFIGURATION = "Release" ]; then
  cp "${base_path}" "$target_path"
  return 0;
fi
{% endhighlight %}

And here is whole shell script:

{% highlight shell %}
#!/bin/sh
export PATH=/opt/local/bin/:/opt/local/sbin:$PATH:/usr/local/bin:

convertPath=`which convert`
echo ${convertPath}
if [[ ! -f ${convertPath} || -z ${convertPath} ]]; then
    echo "WARNING: Skipping Icon versioning, you need to install ImageMagick, you can use brew to simplify process:
    brew install imagemagick"
exit 0;
fi

gsPath=`which gs`
echo ${gsPath}
if [[ ! -f ${gsPath} || -z ${gsPath} ]]; then
    echo "WARNING: Skipping Icon versioning, you need to install ghostscript (fonts) first, you can use brew to simplify process:
    brew install ghostscript"
exit 0;
fi

git=`sh /etc/profile; which git`
branch=`${git} rev-parse --abbrev-ref HEAD`
commits_count=`${git} rev-list ${branch} | wc -l  | tr -d ' '`
commit=`${git} rev-parse --short HEAD`
version=`/usr/libexec/PlistBuddy -c "Print CFBundleVersion" "${INFOPLIST_FILE}"`

DIR="${BASH_SOURCE%/*}"
if [[ ! -d "$DIR" ]]; then DIR="$PWD"; fi

branch="${branch}->${APP_SCHEME_NAME}"

shopt -s extglob
shopt -u extglob
caption="${version} ($commits_count)\n${branch}"
echo $caption

function abspath() { pushd . > /dev/null; if [ -d "$1" ]; then cd "$1"; dirs -l +0; else cd "`dirname \"$1\"`"; cur_dir=`dirs -l +0`; if [ "$cur_dir" == "/" ]; then echo "$cur_dir`basename \"$1\"`"; else echo "$cur_dir/`basename \"$1\"`"; fi; fi; popd > /dev/null; }

function processIcon() {
    base_file=$1
    
    cd "${CONFIGURATION_BUILD_DIR}/${UNLOCALIZED_RESOURCES_FOLDER_PATH}"
    base_path=`find . -name ${base_file}`
    
    real_path=$( abspath "${base_path}" )
    echo "base path ${real_path}"
    
    if [[ ! -f ${base_path} || -z ${base_path} ]]; then
      return;
    fi
    
    # TODO: if they are the same we need to fix it by introducing temp
    target_file=`basename $base_path`
    target_path="${CONFIGURATION_BUILD_DIR}/${UNLOCALIZED_RESOURCES_FOLDER_PATH}/${target_file}"
    
    base_tmp_normalizedFileName="${base_file%.*}-normalized.${base_file##*.}"
    base_tmp_path=`dirname $base_path`
    base_tmp_normalizedFilePath="${base_tmp_path}/${base_tmp_normalizedFileName}"
    
    stored_original_file="${base_tmp_normalizedFilePath}-tmp"
    if [[ -f ${stored_original_file} ]]; then
      echo "found previous file at path ${stored_original_file}, using it as base"
      mv "${stored_original_file}" "${base_path}"
    fi
    
    if [ $CONFIGURATION = "Release" ]; then
      cp "${base_path}" "$target_path"
      return 0;
    fi
    
    echo "Reverting optimized PNG to normal"
    # Normalize
    echo "xcrun -sdk iphoneos pngcrush -revert-iphone-optimizations -q ${base_path} ${base_tmp_normalizedFilePath}"
    xcrun -sdk iphoneos pngcrush -revert-iphone-optimizations -q "${base_path}" "${base_tmp_normalizedFilePath}"
    
    # move original pngcrush png to tmp file
    echo "moving pngcrushed png file at ${base_path} to ${stored_original_file}"
    #rm "$base_path"
    mv "$base_path" "${stored_original_file}"
    
    # Rename normalized png's filename to original one
    echo "Moving normalized png file to original one ${base_tmp_normalizedFilePath} to ${base_path}"
    mv "${base_tmp_normalizedFilePath}" "${base_path}"
    
    width=`identify -format %w ${base_path}`
    height=`identify -format %h ${base_path}`
    band_height=$((($height * 30) / 100))
    band_position=$(($height - $band_height))
    text_position=$(($band_position - 3))
    point_size=$(((13 * $width) / 100))
    
    echo "Image dimensions ($width x $height) - band height $band_height @ $band_position - point size $point_size"
    
    #
    # blur band and text
    #
    convert ${base_path} -blur 10x8 /tmp/blurred.png
    convert /tmp/blurred.png -gamma 0 -fill white -draw "rectangle 0,$band_position,$width,$height" /tmp/mask.png
    convert -size ${width}x${band_height} xc:none -fill 'rgba(0,0,0,0.2)' -draw "rectangle 0,0,$width,$band_height" /tmp/labels-base.png
    convert -background none -size ${width}x${band_height} -pointsize $point_size -fill white -gravity center -gravity South caption:"$caption" /tmp/labels.png
    
    convert ${base_path} /tmp/blurred.png /tmp/mask.png -composite /tmp/temp.png
    
    rm /tmp/blurred.png
    rm /tmp/mask.png
    
    #
    # compose final image
    #
    filename=New${base_file}
    convert /tmp/temp.png /tmp/labels-base.png -geometry +0+$band_position -composite /tmp/labels.png -geometry +0+$text_position -geometry +${w}-${h} -composite "${target_path}"
    
    # clean up
    rm /tmp/temp.png
    rm /tmp/labels-base.png
    rm /tmp/labels.png
    
    echo "Overlayed ${target_path}"
}

icon_count=`/usr/libexec/PlistBuddy -c "Print CFBundleIcons:CFBundlePrimaryIcon:CFBundleIconFiles" "${CONFIGURATION_BUILD_DIR}/${INFOPLIST_PATH}" | wc -l`
last_icon_index=$((${icon_count} - 2))

i=0
while [  $i -lt $last_icon_index ]; do
  icon=`/usr/libexec/PlistBuddy -c "Print CFBundleIcons:CFBundlePrimaryIcon:CFBundleIconFiles:$i" "${CONFIGURATION_BUILD_DIR}/${INFOPLIST_PATH}"`

  if [[ $icon == *.png ]] || [[ $icon == *.PNG ]]
  then
    processIcon $icon
  else
    processIcon "${icon}.png"
    processIcon "${icon}@2x.png"
    processIcon "${icon}@3x.png"
  fi
  let i=i+1
done

# Workaround to fix issue#16 to use wildcard * to actually find the file
# Only 72x72 and 76x76 that we need for ipad app icons
processIcon "AppIcon72x72~ipad*"
processIcon "AppIcon72x72@2x~ipad*"
processIcon "AppIcon76x76~ipad*"
processIcon "AppIcon76x76@2x~ipad*"
{% endhighlight %}

## Result

![Overlayed icons](/static/article-icon-overlaying/02.png)

## Acknowledgments

This is mostly extraction and adaptation of technique from [KZBootstrap](https://github.com/krzysztofzablocki/KZBootstrap) by [Krzysztof Zabłocki](https://twitter.com/merowing_).
