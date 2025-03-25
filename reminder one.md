<p><a target="_blank" href="https://app.eraser.io/workspace/AebbP2LnOMbN75ypnOXg" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

so the csv model how it looks like

```javascript
// here is the reviewcsv one model
file_url: { type: String, required: true },
file_hash: { type: String, required: true, unique: true },
users: [
  {
    email: { type: String, required: true },
    first_sent: { type: Date }, // first invite email
    second_sent: { type: Date }, // reminder one email
    last_sent: { type: Date }, // reminder two email
  },
],
addedBy: { type: Schema.Types.ObjectId, ref: "users", index: true },
pageSettingId: {
  type: Schema.Types.ObjectId,
  ref: "reviewpagesetting",
  index: true,
},

// below is the pagesetting model title: { type: String },
email_invite_template: {
  type: Schema.Types.ObjectId,
  ref: "reviewsinvitetemplate",
  index: true,
},
reminder_email_invite_template: {
  type: Schema.Types.ObjectId,
  ref: "reviewsinvitetemplate",
  index: true,
},
reminder_email_invite_template_last: {
  type: Schema.Types.ObjectId,
  ref: "reviewsinvitetemplate",
  index: true,
},
inviteFor: {
  type: String,
  default: "",
  enum: ["company-with-pos", "product", "company-without-pos"],
},
invite_email_after_travelDate: { type: Number },
send_invite_reminder_firstemail: { type: Boolean, default: false }, // whether to send reminder 1
invite_reminder_after_firstemail: { type: Number }, // how many days after the first email to be sent
send_invite_reminder_secondemail: { type: Boolean, default: false },
last_invite_reminder_after_secondemail: { type: Number },
invited_csv: { type: String },
review_invite_links: [
  {
    link: { type: String },
    linkFor: {
      type: Schema.ObjectId,
      ref: "connectedreviewlocations",
    },
    platform: { type: String },
    posId: {
      type: Schema.Types.ObjectId,
      ref: "integratedpos",
      index: true,
    },
  },
],
reviewInviteSmtpId: {
  type: Schema.Types.ObjectId,
  ref: "smtp",
  index: true,
},
invitedProducts: { type: Array, default: [] },
page_title: { type: String },
background: { type: String },
show_logo: { type: String },
logo: { type: String },
main_image: { type: String },
formfields: [
  {
    size: { type: String },
    type: { type: String },
    name: { type: String },
  },
],
reviewPlatforms: [
  {
    show: { type: Boolean },
    platformId: {
      type: Schema.ObjectId,
      ref: "reviewsplatforms",
    },
  },
],
addedBy: { type: Schema.Types.ObjectId, ref: "users", index: true },
updatedBy: { type: Schema.Types.ObjectId, ref: "users" },
isDeleted: { type: Boolean, default: false },
show_google_widgets: { type: Boolean, default: false },

// now since we dont have access to the reminder here it self so we need to 
// rely on the pagesetting as the admin can change so make sure to implement the 
// reminder email one properly 
async function sendReviewReminderCsvOne() {
  try {
    const reviewCsvs = await db.reviewinvitecsv.find({
      "users.first_sent": { $ne: null }, // Only process users who have received the first email
      "users.second_sent": null, // Only process users who haven't received the second email
      // the reminder does not exist also the admin can stop the reminder in between
      // so we must fallback over the pageSetting collection only
      // reminder: true, // Only process those that have reminders set to true.
    });

    for await (const reviewCsv of reviewCsvs) {
      const pageSetting = await db.reviewpagesetting.findById(
        reviewCsv.pageSettingId,
      );

      if (!pageSetting) {
        console.warn(`Page setting not found for reviewCsv ${reviewCsv._id}`);
        continue;
      }

      if (
        !pageSetting.reminder_email_invite_template ||
        !pageSetting.invite_reminder_after_firstemail
      ) {
        console.warn(
          `Reminder template or first reminder delay not set for page setting ${pageSetting._id}`,
        );
        continue;
      }

      const usersToSendReminder = reviewCsv.users.filter((user) => {
        if (!user.first_sent || user.second_sent) {
          return false; // Skip users without first_sent or with second_sent
        }

        const timeSinceFirstSent = Date.now() - user.first_sent.getTime();
        const reminderDelay =
          pageSetting.invite_reminder_after_firstemail * 24 * 60 * 60 * 1000; // Convert days to milliseconds

        return timeSinceFirstSent >= reminderDelay;
      });

      for await (const user of usersToSendReminder) {
        const emailpayload = {
          templateId: pageSetting.reminder_email_invite_template,
          email: user.email,
          smtpId: pageSetting.reviewInviteSmtpId,
          addedBy: pageSetting.addedBy,
          logo: pageSetting.logo,
        };

        try {
          await ReviewEmails.reviewCsvInvite(emailpayload);
          user.second_sent = new Date();
        } catch (emailError) {
          console.error(
            `Error sending reminder 1 to ${user.email}:`,
            emailError,
          );
        }
      }

      await reviewCsv.save();
    }
  } catch (err) {
    console.error("Error sending review reminder 1:", err);
  }

```




code for the plan controller



```javascript
const db = require("../models");
const constants = require("../utls/constants");
const stripeServices = require("../services/stripeServices");
const stripe = require("stripe")(process.env.STRIPE_KEY);
module.exports = {
  addPlan: async (req, res) => {
    try {
      if (req.body.name) {
        let name = req.body.name;
        req.body.name = name.toLowerCase();
      }

      let findPlan = await db.Plan.findOne({
        where: {
          isDeleted: false,
          name: req.body.name,
        },
      });

      if (findPlan) {
        return res.status(400).json({
          success: false,
          error: {
            code: 400,
            message: constants.PLAN.ALREADY_EXIST,
          },
        });
      }
      let created_product = await stripeServices.create_product({
        name: req.body.name,
      });

      if (created_product) {
        for await (const item of req.body.pricing) {
          var price = await stripe.prices.create({
            product: created_product.id,
            unit_amount: Number(item.unit_amount) * 100,
            currency: "usd",
            recurring: {
              interval: item.interval_time ? item.interval_time : "month",
              interval_count: item.interval_count ? item.interval_count : 1,
            },
          });

          item.stripe_price_id = price.id;
          req.body.stripe_price_id = item.stripe_price_id;
        }
        req.body.addedBy = req.identity.id;
        req.body.stripe_product_id = created_product.id;
        req.body.user_id = req.identity.id;
        let create_subscription_plan = await db.Plan.create(req.body);
        if (create_subscription_plan) {
          return res.status(200).json({
            success: true,
            message: constants.PLAN.CREATED,
            data: create_subscription_plan,
          });
        }
      }

      return res.status(400).json({
        success: false,
        error: {
          code: 400,
          message: constants.PLAN.UNABLE_CREATE_PLAN,
        },
      });
    } catch (err) {
      console.log(err, "===============error");
      return res.status(500).json({
        success: false,
        error: {
          code: 500,
          message: err.message || "Internal server error",
        },
      });
    }
  },
  editPlan: async (req, res) => {
    try {
      let { id } = req.body;
      if (req.body.name) {
        let name = req.body.name;
        req.body.name = name.toLowerCase();
      }
      req.body.updatedAt = new Date();
      await db.Plan.update(req.body, {
        where: { id: id },
      });

      return res.status(200).json({
        success: true,
        message: constants.PLAN.UPDATED,
      });
    } catch (err) {
      console.log(err, "===============error");
      return res.status(500).json({
        success: false,
        error: {
          code: 500,
          message: err.message || "Internal server error",
        },
      });
    }
  },
  listing: async (req, res) => {
    try {
      const { search, page, count, sortBy, status, type } = req.query;

      let where = { isDeleted: false };
      let order = [];

      if (search) {
        where = {
          ...where,
          name: { [db.Sequelize.Op.iLike]: `%${search}%` },
        };
      }

      if (sortBy) {
        const [field, sortType] = sortBy.split(" ");
        order.push([
          field || "createdAt",
          sortType === "desc" ? "DESC" : "ASC",
        ]);
      } else {
        order.push(["createdAt", "DESC"]);
      }

      if (status) {
        where.status = status;
      }
      if (type) {
        where.type = type;
      }

      let options = {
        where,
        order,
        include: [
          {
            model: db.Feature,
            as: "features",
          },
        ],
      };

      if (page && count) {
        options.offset = (Number(page) - 1) * Number(count);
        options.limit = Number(count);
      }

      const { count: total, rows: result } =
        await db.Plan.findAndCountAll(options);

      return res.status(200).json({
        success: true,
        data: result,
        total: total,
      });
    } catch (err) {
      return res.status(500).json({
        success: false,
        error: {
          code: 500,
          message: err.message || "Internal server error",
        },
      });
    }
  },
  getPlanById: async (req, res) => {
    try {
      let { id } = req.query;

      if (!id) {
        return res.status(400).json({
          success: false,
          error: {
            code: 400,
            message: constants.PLAN.ID_REQUIRED,
          },
        });
      }

      const detail = await db.Plan.findOne({
        where: {
          id: id,
          isDeleted: false,
        },
        include: [
          {
            model: db.Feature,
            as: "features",
          },
        ],
      });

      if (!detail) {
        return res.status(404).json({
          success: false,
          error: {
            code: 404,
            message: constants.PLAN.NOT_FOUND,
          },
        });
      }

      return res.status(200).json({
        success: true,
        data: detail,
      });
    } catch (err) {
      return res.status(500).json({
        success: false,
        error: {
          code: 500,
          message: err.message || "Internal server error",
        },
      });
    }
  },
  deletePlan: async (req, res) => {
    try {
      const { id } = req.query;

      if (!id) {
        return res.status(400).json({
          success: false,
          message: constants.PLAN.ID_REQUIRED,
        });
      }
      const planData = await db.Plan.findOne({
        where: {
          id: id,
          isDeleted: false,
        },
      });
      if (!planData) {
        return res.status(404).json({
          success: false,
          message: constants.PLAN.NOT_FOUND,
        });
      }

      await db.Plan.update({ isDeleted: true }, { where: { id: id } });

      return res.status(200).json({
        success: true,
        message: constants.PLAN.DELETED,
      });
    } catch (err) {
      return res.status(500).json({
        success: false,
        error: {
          code: 500,
          message: err.message || "Internal server error",
        },
      });
    }
  },
  changeStatus: async (req, res) => {
    try {
      const { id, status } = req.body;

      if (!id) {
        return res.status(400).json({
          success: false,
          error: {
            code: 400,
            message: constants.PLAN.ID_REQUIRED,
          },
        });
      }

      if (status === undefined || status === null) {
        return res.status(400).json({
          success: false,
          error: {
            code: 400,
            message: constants.PLAN.STATUS_REQUIRED,
          },
        });
      }

      const planData = await db.Plan.findOne({
        where: { id: id },
      });
      if (!planData) {
        return res.status(404).json({
          success: false,
          error: {
            code: 404,
            message: constants.PLAN.NOT_FOUND,
          },
        });
      }

      await db.Plan.update({ status }, { where: { id: id } });

      return res.status(200).json({
        success: true,
        message: constants.PLAN.STATUS_CHANGED,
      });
    } catch (err) {
      return res.status(500).json({
        success: false,
        error: {
          code: 500,
          message: err.message || "Internal server error",
        },
      });
    }
  },
};
```




code for the subscription controller

```javascript
"use strict";

const db = require("../models");
const constants = require("../utls/constants");
const { Op } = require("sequelize");

module.exports = {
  /**Add multiple features */
  addMultipleFeatures: async (req, res) => {
    try {
      const features = req.body.features;

      if (!features || !Array.isArray(features)) {
        return res.status(400).json({
          success: false,
          error: {
            code: 400,
            message: constants.onBoarding.PAYLOAD_MISSING,
          },
        });
      }

      const createdFeatures = [];

      for (const feature of features) {
        feature.createdAt = new Date();
        feature.updatedAt = new Date();
        feature.isDeleted = false;

        const isExist = await db.Feature.findOne({
          where: {
            name: feature.name,
            isDeleted: false,
          },
        });

        if (isExist) {
          continue;
        }

        try {
          const createdFeature = await db.Feature.create(feature);
          createdFeatures.push(createdFeature);
        } catch (err) {
          console.log(err);
        }
      }

      return res.status(200).json({
        success: true,
        message:
          createdFeatures.length > 0
            ? constants.FEATURE.CREATED
            : constants.FEATURE.NO_NEW_FEATURES,
        data: createdFeatures,
      });
    } catch (err) {
      console.log(err);
      return res.status(400).json({
        success: false,
        error: {
          code: 400,
          message: "Error: " + err.message,
        },
      });
    }
  },
  /** For update feature */
  updateFeature: async (req, res) => {
    try {
      let id = req.body.id;
      if (!id) {
        return res.status(400).json({
          success: false,
          error: {
            code: 400,
            message: constants.onBoarding.PAYLOAD_MISSING,
          },
        });
      }

      const Feature = await db.Feature.findOne({
        where: { id: id },
      });

      if (!Feature) {
        return res.status(404).json({
          success: false,
          error: {
            code: "404",
            message: constants.FEATURE.NOT_FOUND,
          },
        });
      }
      const data = req.body;

      const updatedFeature = await db.Feature.update(data, {
        where: { id: id },
      });

      return res.status(200).json({
        success: true,
        message: constants.FEATURE.UPDATED,
      });
    } catch (err) {
      return res.status(400).json({
        success: false,
        error: {
          code: 400,
          message: "" + err,
        },
      });
    }
  },
  activateDeactivateFeature: async (req, res) => {
    try {
      let id = req.body.id;
      if (!id) {
        return res.status(400).json({
          success: false,
          error: {
            code: 400,
            message: constants.onBoarding.PAYLOAD_MISSING,
          },
        });
      }
      const Feature = await db.Feature.findOne({
        where: { id: id },
      });
      if (!Feature) {
        return res.status(404).json({
          success: false,
          message: constants.FEATURE.NOT_FOUND,
        });
      }
      const query = {
        status: "active",
      };
      let message;
      let message1;
      if (Feature.status == "active") {
        query.status = "deactive";
        message = constants.FEATURE.STATUS_CHANGED;
      } else {
        query.status = "active";
        message1 = constants.FEATURE.STATUS_CHANGED;
      }
      const updateData = await db.Feature.update(query, {
        where: { id: id },
      });
      if (updateData) {
        return res.status(200).json({
          success: true,
          message: Feature.status === "deactive" ? message1 : message,
        });
      }
    } catch (err) {
      return res.status(400).json({
        success: false,
        error: {
          code: 400,
          message: "" + err,
        },
      });
    }
  },
  /** for get list of Feature */
  getAllFeaturesList: async (req, res) => {
    try {
      let { search, sortBy, page, count, status } = req.query;
      let where = {};
      let order = [];

      if (search) {
        where.name = {
          [Op.iLike]: `%${search}%`,
        };
      }

      where.isDeleted = false;

      if (sortBy) {
        let [field, sortType] = sortBy.split(" ");
        order.push([
          field || "createdAt",
          sortType === "desc" ? "DESC" : "ASC",
        ]);
      } else {
        order.push(["createdAt", "DESC"]);
      }

      if (status) {
        where.status = status;
      }

      const options = {
        attributes: ["status", "name", "createdAt", "updatedAt", "isDeleted"],
        where,
        order,
      };

      if (page && count) {
        options.offset = (Number(page) - 1) * Number(count);
        options.limit = Number(count);
      }

      const { rows, count: total } = await db.Feature.findAndCountAll(options);

      return res.status(200).json({
        success: true,
        data: rows,
        total: total,
      });
    } catch (err) {
      return res.status(500).json({
        success: false,
        error: {
          code: 400,
          message: "" + err,
        },
      });
    }
  },
  /** for delete Feature */
  deleteFeature: async (req, res) => {
    try {
      let id = req.query.id;
      if (!id) {
        return res.status(400).json({
          success: false,
          error: {
            code: 400,
            message: constants.onBoarding.PAYLOAD_MISSING,
          },
        });
      }
      const Feature = await db.Feature.findOne({
        where: { id: id },
      });
      if (!Feature) {
        return res.status(404).json({
          success: false,
          error: {
            code: "404",
            message: constants.FEATURE.NOT_FOUND,
          },
        });
      }

      const updateFeature = await db.Feature.update(
        { isDeleted: true },
        { where: { id: id } },
      );

      return res.status(200).json({
        success: true,
        message: constants.FEATURE.DELETED,
      });
    } catch (err) {
      return res.status(400).json({
        success: false,
        error: {
          code: 400,
          message: "" + err,
        },
      });
    }
  },
  /** for get Fetaure Post detail */
  getFeatureDetail: async (req, res) => {
    try {
      const id = req.query.id;
      if (!id) {
        return res.status(401).json({
          success: false,
          error: {
            code: 401,
            message: constants.onBoarding.PAYLOAD_MISSING,
          },
        });
      }
      const getFeature = await db.Feature.findByPk(id);
      if (getFeature) {
        return res.status(200).json({
          success: true,
          message: constants.FEATURE.RETRIEVED,
          payload: getFeature,
        });
      } else {
        return res.status(404).json({
          success: false,
          error: {
            code: 404,
            message: constants.FEATURE.NOT_FOUND,
          },
        });
      }
    } catch (err) {
      return res.status(400).json({
        success: false,
        error: {
          code: 400,
          message: "" + err,
        },
      });
    }
  },
  getAllSubscriptions: async (req, res) => {
    try {
        const { count, page, search, status, userId, sortBy } = req.query;

        // Base query options
        const queryOptions = {
            include: [
                {
                    model: db.User,
                    as: 'userDetail',
                    attributes: ['id', 'fullName'], // Include only necessary fields
                    required: false, // Use LEFT JOIN
                },
                {
                    model: db.Plan,
                    as: 'planDetails',
                    attributes: ['id', 'name'], // Include only necessary fields
                    required: false, // Use LEFT JOIN
                },
            ],
            where: {}, // Initialize empty where clause
            order: [['createdAt', 'DESC']], // Default sorting
        };

        // Add search filter
        if (search) {
            queryOptions.where[Op.or] = [
                { '$userDetail.fullName$': { [Op.like]: `%${search}%` } },
                { '$planDetails.name$': { [Op.like]: `%${search}%` } },
            ];
        }

        // Add status filter
        if (status) {
            queryOptions.where.status = status;
        }

        // Add user ID filter
        if (userId) {
            queryOptions.where.user_id = userId;
        }

        // Add sorting
        if (sortBy) {
            const [field, sortType] = sortBy.split(" ");
            queryOptions.order = [[field || 'createdAt', sortType === 'desc' ? 'DESC' : 'ASC']];
        }

        // Count total records (for pagination)
        const total = await db.Subscription.count({
            where: queryOptions.where,
            include: queryOptions.include,
        });

        // Add pagination
        if (page && count) {
            const offset = (Number(page) - 1) * Number(count);
            queryOptions.offset = offset;
            queryOptions.limit = Number(count);
        }

        // Fetch subscriptions with filters and pagination
        const result = await db.Subscription.findAll(queryOptions);

        return res.status(200).json({
            success: true,
            data: result,
            total: total,
        });
    } catch (err) {
        console.error("Error in getAllSubscriptions:", err);
        return res.status(500).json({
            success: false,
            error: { code: 500, message: err.message || "An error occurred" },
        });
    }
},
};


```




<!--- Eraser file: https://app.eraser.io/workspace/AebbP2LnOMbN75ypnOXg --->