## yii active form 根据model的rules生成js规则
> demo
根据model的规则，会为name字段生成一个required的js验证，接下来根据这个demo来追踪这个js验证生成的步骤
```
<?php $form = ActiveForm::begin([
    'action'=>'xxx',
])
?>
<?= $form->field($model, 'name')->input('input')?>
<?php ActiveForm::end()?>
```
```
class TestModel extends Model
{
    public $name;
    public function rules()
    {
        return [
            ['name', 'required']
        ];
    }

    public function attributeLabels()
    {
        return [
            'name'=>'名字',
        ];
    }
}
```
> yii\widgets\ActiveForm field方法,生成了一个yii\widgets\ActiveField对象
```
public function field($model, $attribute, $options = []) {
    $config = $this->fieldConfig;
    if ($config instanceof \Closure) {
        $config = call_user_func($config, $model, $attribute);
    }
    if (!isset($config['class'])) {
        $config['class'] = $this->fieldClass;
    }
    return Yii::createObject(ArrayHelper::merge($config, $options, [
        'model' => $model,
        'attribute' => $attribute,
        'form' => $this,
    ]));
}
```
> 调用yii\widgets\ActiveField的input方法,生成一个input，然后写入到$this->parts,并且返回ActiveField的对象
```
public function input($type, $options = []) {
    $options = array_merge($this->inputOptions, $options);
    $this->adjustLabelFor($options);
    $this->parts['{input}'] = Html::activeInput($type, $this->model, $this->attribute, $options);

    return $this;
}
```
> 输出ActiveField对象的时候会调用__toString的方法
```
<?= $form->field($model, 'name')->input('input')?>
```
```
public function __toString() {
    // __toString cannot throw exception
    // use trigger_error to bypass this limitation
    try {
        return $this->render();
    } catch (\Exception $e) {
        ErrorHandler::convertExceptionToError($e);
        return '';
    }
}
```
> 调用render
```
public function render($content = null) {
    if ($content === null) {
        if (!isset($this->parts['{input}'])) {
            $this->textInput();
        }
        if (!isset($this->parts['{label}'])) {
            $this->label();
        }
        if (!isset($this->parts['{error}'])) {
            $this->error();
        }
        if (!isset($this->parts['{hint}'])) {
            $this->hint(null);
        }
        $content = strtr($this->template, $this->parts);
    } elseif (!is_string($content)) {
        $content = call_user_func($content, $this);
    }

    return $this->begin() . "\n" . $content . "\n" . $this->end();
}
```
```
public function begin() {
    if ($this->form->enableClientScript) {
        $clientOptions = $this->getClientOptions();
        if (!empty($clientOptions)) {
            $this->form->attributes[] = $clientOptions;
        }
    }

    $inputID = $this->getInputId();
    $attribute = Html::getAttributeName($this->attribute);
    $options = $this->options;
    $class = isset($options['class']) ? [$options['class']] : [];
    $class[] = "field-$inputID";
    if ($this->model->isAttributeRequired($attribute)) {
        $class[] = $this->form->requiredCssClass;
    }
    if ($this->model->hasErrors($attribute)) {
        $class[] = $this->form->errorCssClass;
    }
    $options['class'] = implode(' ', $class);
    $tag = ArrayHelper::remove($options, 'tag', 'div');

    return Html::beginTag($tag, $options);
}
```
```
public function end() {
    return Html::endTag(ArrayHelper::keyExists('tag', $this->options) ? $this->options['tag'] : 'div');
}
```
> 在begin方法里面调用了getClientOptions方法
```
protected function getClientOptions() {
    $attribute = Html::getAttributeName($this->attribute);
    if (!in_array($attribute, $this->model->activeAttributes(), true)) {
        return [];
    }

    $enableClientValidation = $this->enableClientValidation || $this->enableClientValidation === null && $this->form->enableClientValidation;
    $enableAjaxValidation = $this->enableAjaxValidation || $this->enableAjaxValidation === null && $this->form->enableAjaxValidation;

    if ($enableClientValidation) {
        $validators = [];
        foreach ($this->model->getActiveValidators($attribute) as $validator) {
            /* @var $validator \yii\validators\Validator */
            $js = $validator->clientValidateAttribute($this->model, $attribute, $this->form->getView());
            if ($validator->enableClientValidation && $js != '') {
                if ($validator->whenClient !== null) {
                    $js = "if (({$validator->whenClient})(attribute, value)) { $js }";
                }
                $validators[] = $js;
            }
        }
    }

    if (!$enableAjaxValidation && (!$enableClientValidation || empty($validators))) {
        return [];
    }

    $options = [];

    $inputID = $this->getInputId();
    $options['id'] = Html::getInputId($this->model, $this->attribute);
    $options['name'] = $this->attribute;

    $options['container'] = isset($this->selectors['container']) ? $this->selectors['container'] : ".field-$inputID";
    $options['input'] = isset($this->selectors['input']) ? $this->selectors['input'] : "#$inputID";
    if (isset($this->selectors['error'])) {
        $options['error'] = $this->selectors['error'];
    } elseif (isset($this->errorOptions['class'])) {
        $options['error'] = '.' . implode('.', preg_split('/\s+/', $this->errorOptions['class'], -1, PREG_SPLIT_NO_EMPTY));
    } else {
        $options['error'] = isset($this->errorOptions['tag']) ? $this->errorOptions['tag'] : 'span';
    }

    $options['encodeError'] = !isset($this->errorOptions['encode']) || $this->errorOptions['encode'];
    if ($enableAjaxValidation) {
        $options['enableAjaxValidation'] = true;
    }
    foreach (['validateOnChange', 'validateOnBlur', 'validateOnType', 'validationDelay'] as $name) {
        $options[$name] = $this->$name === null ? $this->form->$name : $this->$name;
    }

    if (!empty($validators)) {
        $options['validate'] = new JsExpression("function (attribute, value, messages, deferred, \$form) {" . implode('', $validators) . '}');
    }

    // only get the options that are different from the default ones (set in yii.activeForm.js)
    return array_diff_assoc($options, [
        'validateOnChange' => true,
        'validateOnBlur' => true,
        'validateOnType' => false,
        'validationDelay' => 500,
        'encodeError' => true,
        'error' => '.help-block',
    ]);
}
```
> 根据$model的rules来获取到name的validator,得到RequiredValidator
```
foreach ($this->model->getActiveValidators($attribute) as $validator) {
    /* @var $validator \yii\validators\Validator */
    $js = $validator->clientValidateAttribute($this->model, $attribute, $this->form->getView());
    if ($validator->enableClientValidation && $js != '') {
        if ($validator->whenClient !== null) {
            $js = "if (({$validator->whenClient})(attribute, value)) { $js }";
        }
        $validators[] = $js;
    }
}
```
> 调用RequiredValidator的clientValidateAttribute返回yii.validation.required(value, messages, {"message":"名字不能为空。"});
```
public function clientValidateAttribute($model, $attribute, $view) {
    $options = [];
    if ($this->requiredValue !== null) {
        $options['message'] = Yii::$app->getI18n()->format($this->message, [
            'requiredValue' => $this->requiredValue,
        ], Yii::$app->language);
        $options['requiredValue'] = $this->requiredValue;
    } else {
        $options['message'] = $this->message;
    }
    if ($this->strict) {
        $options['strict'] = 1;
    }

    $options['message'] = Yii::$app->getI18n()->format($options['message'], [
        'attribute' => $model->getAttributeLabel($attribute),
    ], Yii::$app->language);

    ValidationAsset::register($view);

    return 'yii.validation.required(value, messages, ' . json_encode($options, JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE) . ');';
}
```
> 将返回来的js
```
$validators[] = $js;
```
```
$options['validate'] = new JsExpression("function (attribute, value, messages, deferred, \$form) {" . implode('', $validators) . '}');
```
> getClientOptions返回下面的数据到begin里面,并且写入到ActiveForm的attributes
```
array (size=1)
  0 => 
    array (size=5)
      'id' => string 'testmodel-name' (length=14)
      'name' => string 'name' (length=4)
      'container' => string '.field-testmodel-name' (length=21)
      'input' => string '#testmodel-name' (length=15)
      'validate' => 
        object(yii\web\JsExpression)[139]
          public 'expression' => string 'function (attribute, value, messages, deferred, $form) {yii.validation.required(value, messages, {"message":"名字不能为空。"});}' (length=135)
```
```
$this->form->attributes[] = $clientOptions;
```
> ActiveForm::end()会用调用run()方法
```
public function run() {
    if (!empty($this->_fields)) {
        throw new InvalidCallException('Each beginField() should have a matching endField() call.');
    }

    $content = ob_get_clean();
    echo Html::beginForm($this->action, $this->method, $this->options);
    echo $content;

    if ($this->enableClientScript) {
        $id = $this->options['id'];
        $options = Json::htmlEncode($this->getClientOptions());
        $attributes = Json::htmlEncode($this->attributes);
        $view = $this->getView();
        ActiveFormAsset::register($view);
        $view->registerJs("jQuery('#$id').yiiActiveForm($attributes, $options);");
    }

    echo Html::endForm();
}
```
> 将js注入到页面中
```
$view->registerJs("jQuery('#$id').yiiActiveForm($attributes, $options);");
```
> 调用yii.activeForm.js的yiiActiveForm方法
```
$.fn.yiiActiveForm = function (method) {
    if (methods[method]) {
        return methods[method].apply(this, Array.prototype.slice.call(arguments, 1));
    } else if (typeof method === 'object' || !method) {
        return methods.init.apply(this, arguments);
    } else {
        $.error('Method ' + method + ' does not exist on jQuery.yiiActiveForm');
        return false;
    }
};
```
> 调用methods的init方法,并且this是jQuery('#$id')
```
init: function (attributes, options) {
    return this.each(function () {
        var $form = $(this);
        if ($form.data('yiiActiveForm')) {
            return;
        }

        var settings = $.extend({}, defaults, options || {});
        if (settings.validationUrl === undefined) {
            settings.validationUrl = $form.attr('action');
        }

        $.each(attributes, function (i) {
            attributes[i] = $.extend({value: getValue($form, this)}, attributeDefaults, this);
            watchAttribute($form, attributes[i]);
        });

        $form.data('yiiActiveForm', {
            settings: settings,
            attributes: attributes,
            submitting: false,
            validated: false,
            options: getFormOptions($form)
        });

        /**
         * Clean up error status when the form is reset.
         * Note that $form.on('reset', ...) does work because the "reset" event does not bubble on IE.
         */
        $form.bind('reset.yiiActiveForm', methods.resetForm);

        if (settings.validateOnSubmit) {
            $form.on('mouseup.yiiActiveForm keyup.yiiActiveForm', ':submit', function () {
                $form.data('yiiActiveForm').submitObject = $(this);
            });
            $form.on('submit.yiiActiveForm', methods.submitForm);
        }
        var event = $.Event(events.afterInit);
        $form.trigger(event);
    });
}
```
> 遍历form的attributes的调用watchAttribute
```
attributes[i] = $.extend({value: getValue($form, this)}, attributeDefaults, this);
watchAttribute($form, attributes[i]);
```
> 为每个需要验证的元素绑定blur,change,keyup事件
```
var watchAttribute = function ($form, attribute) {
    var $input = findInput($form, attribute);
    if (attribute.validateOnChange) {
        $input.on('change.yiiActiveForm', function () {
            validateAttribute($form, attribute, false);
        });
    }
    if (attribute.validateOnBlur) {
        $input.on('blur.yiiActiveForm', function () {
            if (attribute.status == 0 || attribute.status == 1) {
                validateAttribute($form, attribute, true);
            }
        });
    }
    if (attribute.validateOnType) {
        $input.on('keyup.yiiActiveForm', function (e) {
            if ($.inArray(e.which, [16, 17, 18, 37, 38, 39, 40]) !== -1 ) {
                return;
            }
            if (attribute.value !== getValue($form, attribute)) {
                validateAttribute($form, attribute, false, attribute.validationDelay);
            }
        });
    }
};
```
> 绑定validateAttribute
```
var validateAttribute = function ($form, attribute, forceValidate, validationDelay) {
    var data = $form.data('yiiActiveForm');

    if (forceValidate) {
        attribute.status = 2;
    }
    $.each(data.attributes, function () {
        if (this.value !== getValue($form, this)) {
            this.status = 2;
            forceValidate = true;
        }
    });
    if (!forceValidate) {
        return;
    }

    if (data.settings.timer !== undefined) {
        clearTimeout(data.settings.timer);
    }
    data.settings.timer = setTimeout(function () {
        if (data.submitting || $form.is(':hidden')) {
            return;
        }
        $.each(data.attributes, function () {
            if (this.status === 2) {
                this.status = 3;
                $form.find(this.container).addClass(data.settings.validatingCssClass);
            }
        });
        methods.validate.call($form);
    }, validationDelay ? validationDelay : 200);
};
```
> 调用methods.validate.call($form)进行验证
```
validate: function (forceValidate) {
    if (forceValidate) {
        $(this).data('yiiActiveForm').submitting = true;
    }

    var $form = $(this),
        data = $form.data('yiiActiveForm'),
        needAjaxValidation = false,
        messages = {},
        deferreds = deferredArray(),
        submitting = data.submitting;

    if (submitting) {
        var event = $.Event(events.beforeValidate);
        $form.trigger(event, [messages, deferreds]);

        if (event.result === false) {
            data.submitting = false;
            submitFinalize($form);
            return;
        }
    }

    // client-side validation
    $.each(data.attributes, function () {
        this.$form = $form;
        if (!$(this.input).is(":disabled")) {
            this.cancelled = false;
            // perform validation only if the form is being submitted or if an attribute is pending validation
            if (data.submitting || this.status === 2 || this.status === 3) {
                var msg = messages[this.id];
                if (msg === undefined) {
                    msg = [];
                    messages[this.id] = msg;
                }
                var event = $.Event(events.beforeValidateAttribute);
                $form.trigger(event, [this, msg, deferreds]);
                if (event.result !== false) {
                    if (this.validate) {
                        this.validate(this, getValue($form, this), msg, deferreds, $form);
                    }
                    if (this.enableAjaxValidation) {
                        needAjaxValidation = true;
                    }
                } else {
                    this.cancelled = true;
                }
            }
        }
    });

    // ajax validation
    $.when.apply(this, deferreds).always(function() {
        // Remove empty message arrays
        for (var i in messages) {
            if (0 === messages[i].length) {
                delete messages[i];
            }
        }
        if (needAjaxValidation && ($.isEmptyObject(messages) || data.submitting)) {
            var $button = data.submitObject,
                extData = '&' + data.settings.ajaxParam + '=' + $form.attr('id');
            if ($button && $button.length && $button.attr('name')) {
                extData += '&' + $button.attr('name') + '=' + $button.attr('value');
            }
            $.ajax({
                url: data.settings.validationUrl,
                type: $form.attr('method'),
                data: $form.serialize() + extData,
                dataType: data.settings.ajaxDataType,
                complete: function (jqXHR, textStatus) {
                    $form.trigger(events.ajaxComplete, [jqXHR, textStatus]);
                },
                beforeSend: function (jqXHR, settings) {
                    $form.trigger(events.ajaxBeforeSend, [jqXHR, settings]);
                },
                success: function (msgs) {
                    if (msgs !== null && typeof msgs === 'object') {
                        $.each(data.attributes, function () {
                            if (!this.enableAjaxValidation || this.cancelled) {
                                delete msgs[this.id];
                            }
                        });
                        updateInputs($form, $.extend(messages, msgs), submitting);
                    } else {
                        updateInputs($form, messages, submitting);
                    }
                },
                error: function () {
                    data.submitting = false;
                    submitFinalize($form);
                }
            });
        } else if (data.submitting) {
            // delay callback so that the form can be submitted without problem
            setTimeout(function () {
                updateInputs($form, messages, submitting);
            }, 200);
        } else {
            updateInputs($form, messages, submitting);
        }
    });
},
```
> this.validate(this, getValue($form, this), msg, deferreds, $form);调用从后台生成的这段json里面的validate
```
array (size=5)
      'id' => string 'testmodel-name' (length=14)
      'name' => string 'name' (length=4)
      'container' => string '.field-testmodel-name' (length=21)
      'input' => string '#testmodel-name' (length=15)
      'validate' => 
        object(yii\web\JsExpression)[139]
          public 'expression' => string 'function (attribute, value, messages, deferred, $form) {yii.validation.required(value, messages, {"message":"名字不能为空。"});}' (length=135)
```
```
function (attribute, value, messages, deferred, $form) {yii.validation.required(value, messages, {"message":"名字不能为空。"});}
```
> 调用yii.validation.js的required方法,进行具体的验证
```
required: function (value, messages, options) {
    var valid = false;
    if (options.requiredValue === undefined) {
        var isString = typeof value == 'string' || value instanceof String;
        if (options.strict && value !== undefined || !options.strict && !pub.isEmpty(isString ? $.trim(value) : value)) {
            valid = true;
        }
    } else if (!options.strict && value == options.requiredValue || options.strict && value === options.requiredValue) {
        valid = true;
    }

    if (!valid) {
        pub.addMessage(messages, options.message, value);
    }
}
```
> 如果有错,messages的信息会同步回来,调用updateInputs($form, messages, submitting);
```
var updateInputs = function ($form, messages, submitting) {
    var data = $form.data('yiiActiveForm');

    if (data === undefined) {
        return false;
    }

    if (submitting) {
        var errorAttributes = [];
        $.each(data.attributes, function () {
            if (!$(this.input).is(":disabled") && !this.cancelled && updateInput($form, this, messages)) {
                errorAttributes.push(this);
            }
        });

        $form.trigger(events.afterValidate, [messages, errorAttributes]);

        updateSummary($form, messages);

        if (errorAttributes.length) {
            if (data.settings.scrollToError) {
                var top = $form.find($.map(errorAttributes, function(attribute) {
                    return attribute.input;
                }).join(',')).first().closest(':visible').offset().top;
                var wtop = $(window).scrollTop();
                if (top < wtop || top > wtop + $(window).height()) {
                    $(window).scrollTop(top);
                }
            }
            data.submitting = false;
        } else {
            data.validated = true;
            if (data.submitObject) {
                data.submitObject.trigger("click");
            } else {
                $form.submit();
            }
        }
    } else {
        $.each(data.attributes, function () {
            if (!this.cancelled && (this.status === 2 || this.status === 3)) {
                updateInput($form, this, messages);
            }
        });
    }
    submitFinalize($form);
};
```
> 调用updateInput
```
var updateInput = function ($form, attribute, messages) {
    var data = $form.data('yiiActiveForm'),
        $input = findInput($form, attribute),
        hasError = false;

    if (!$.isArray(messages[attribute.id])) {
        messages[attribute.id] = [];
    }
    $form.trigger(events.afterValidateAttribute, [attribute, messages[attribute.id]]);

    attribute.status = 1;
    if ($input.length) {
        hasError = messages[attribute.id].length > 0;
        var $container = $form.find(attribute.container);
        var $error = $container.find(attribute.error);
        if (hasError) {
            if (attribute.encodeError) {
                $error.text(messages[attribute.id][0]);
            } else {
                $error.html(messages[attribute.id][0]);
            }
            $container.removeClass(data.settings.validatingCssClass + ' ' + data.settings.successCssClass)
                .addClass(data.settings.errorCssClass);
        } else {
            $error.empty();
            $container.removeClass(data.settings.validatingCssClass + ' ' + data.settings.errorCssClass + ' ')
                .addClass(data.settings.successCssClass);
        }
        attribute.value = getValue($form, attribute);
    }
    return hasError;
};
```
> check error
通过下面语句知道有没有错,如果有错,就写入到$error结构里面去
```
hasError = messages[attribute.id].length > 0;
```
```
$error.text(messages[attribute.id][0]);
```
> 回到init方法里面,继续往下看
```
$form.data('yiiActiveForm', {
    settings: settings,
    attributes: attributes,
    submitting: false,
    validated: false,
    options: getFormOptions($form)
});

/**
 * Clean up error status when the form is reset.
 * Note that $form.on('reset', ...) does work because the "reset" event does not bubble on IE.
 */
$form.bind('reset.yiiActiveForm', methods.resetForm);

if (settings.validateOnSubmit) {
    $form.on('mouseup.yiiActiveForm keyup.yiiActiveForm', ':submit', function () {
        $form.data('yiiActiveForm').submitObject = $(this);
    });
    $form.on('submit.yiiActiveForm', methods.submitForm);
}
var event = $.Event(events.afterInit);
$form.trigger(event);
```
> form submit的时候调用submitForm的方法
```
submitForm: function () {
    var $form = $(this),
        data = $form.data('yiiActiveForm');

    if (data.validated) {
        // Second submit's call (from validate/updateInputs)
        data.submitting = false;
        var event = $.Event(events.beforeSubmit);
        $form.trigger(event);
        if (event.result === false) {
            data.validated = false;
            submitFinalize($form);
            return false;
        }
        updateHiddenButton($form);
        return true;   // continue submitting the form since validation passes
    } else {
        // First submit's call (from yii.js/handleAction) - execute validating
        setSubmitFinalizeDefer($form);

        if (data.settings.timer !== undefined) {
            clearTimeout(data.settings.timer);
        }
        data.submitting = true;
        methods.validate.call($form);
        return false;
    }
},
```
> 调用data.submitting = true;并且methods.validate.call($form);在验证通过之后在updateInputs之后调用submit方法
```
if (submitting) {
    var errorAttributes = [];
    $.each(data.attributes, function () {
        if (!$(this.input).is(":disabled") && !this.cancelled && updateInput($form, this, messages)) {
            errorAttributes.push(this);
        }
    });

    $form.trigger(events.afterValidate, [messages, errorAttributes]);

    updateSummary($form, messages);

    if (errorAttributes.length) {
        if (data.settings.scrollToError) {
            var top = $form.find($.map(errorAttributes, function(attribute) {
                return attribute.input;
            }).join(',')).first().closest(':visible').offset().top;
            var wtop = $(window).scrollTop();
            if (top < wtop || top > wtop + $(window).height()) {
                $(window).scrollTop(top);
            }
        }
        data.submitting = false;
    } else {
        data.validated = true;
        if (data.submitObject) {
            data.submitObject.trigger("click");
        } else {
            $form.submit();
        }
    }
}
```
> 到此yiiactiveform前端js验证生产流程源码解读至此结束